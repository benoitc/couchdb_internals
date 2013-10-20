The replication algorithm in pseudo code.

## Goal

## The replication algorithm in details

Pseudo-code extracted from the [couch_replicator](https://github.com/refuge/couch_core/tree/master/apps/couch_replicator) application in [rcouch](http://rcouch.org).

###0. start

Parse the replicator document
Calculate a replication id for this replication task


###1. init

(when the replication task is created)

 <code>
    Get source infos (/source)
    Get target info

    SourceSeq = update_seq in source infos

    find replication logs:
        logs are stored in checkpoints document
        checkpoint document id is defined by `logid$

        logid = _local/replication id
        get replications logs on [Source, Target]


    compare replication logs between Source and target:

        if Source.`sessionid` == Target.`sessionid`:
            """ records have the same session id, then we have a valid
            replication history """

            StartSeq = Source.`source_last_seq` or 0
            History = Source.`history`

        else:
            """ scanning histories to find differences """
            compare replication history between Source and Target:

                An history is a list of committed replications.
                we iterate over each elements side by side:

                    while source != [] and target != []:
                        S0, SourceRest = source pop left
                        T0, TargetRest = target pop left

                        if Target has S0.`sessionid`:
                            StartSeq = S0.`recorded_seq` or 0
                            History = SourceRest

                        else:
                            if SourceRest has Target.`sessionid`:
                                StartSeq = T0.`recorded_seq` or 0
                                History = TargetRest

                        source = SourceRest
                        target = TargetRest

                    if not StartSeq:
                        """ no common ancestors found """

                        StartSeq = 0
                        History = []


        return StartSeq, History

</code>

> note: Does the replicator try to find old checkpoint documents by testing the old
> document id is, then changing to new id?

###2. read changes on source since StartSeq

<code>
    GET /_changes or subscribe to local changes

    params:
        since
        style: 'all_docs'
        feed : 'normal' or 'continuous' if continuous=true
        heartbeat

        if filter:
            pass filter params

</code>

> **note**: when reading changes make sure the doc we receive has an id. if not
>ignore it. (some versions of couchdb had a bug)

###3. on change

<code>
    find missing on target:
        get all pair {Id, Revs} and revscount:

            idrevs = []
            revscount = 0
            for doc in changes:
                idrevs += [{Id, Revs}]
                revscount += len(Revs)

        get missing revs on target with idrevs:
            if target is remote:
                convert revs to rev str
                body = {id:revsttr, ...}

                POST /sourcedb/_revs_diff jsonbody
            else:
                get local missing revs

            return list of {Id, MissingRevs, PossibleAncestors} as idsrevs

        get missing count
            missingcount = 0
            for {Id, MissingRevs, PossibleAncestors} in missingrevs:
                missingcount += len(MissingRevs)


        stats = {revscount, missingcount}
        return {idsrevs, stats}

</code>


###4. process changes:

<code>
    if source is local:
        for {Id, MissingRevs, PossibleAncestors} in idsrevs:
            Options = {atts_since, PossibleAncestors}, latest]
            fetch doc:
                Doc = open_doc_revs(db, Id, MissingRevs, Options)

            maybe flush doc:
                # update doc use new_edits=alse function to reuse the same
                # revision and not edit the documen
                updatd doc if nb doc <= worker_batch_size and still changes
                to process
                    else wait next changes
    else:
        spawn a process to fetch a doc
        Use multipart API:

        ---- boundary, name=doc
        ....
        ---- boundary, name=attname
        ...

        once read ->
            maybe flush doc
</code>

###5. update state

Once the documents are stored on the target, the state is updated. The
checkpoint will be updated with this state at the given interval

###6. listen on other changes

If the replication is continuous, wait for other changes.

## How to calculate a replication ID?

There have been 3 versions of the CouchDB replication protocol that can be
detcted using the replicaton ID.

The current version of the replication ID is calculated following such rules:

<code>
    base = [node uuid, src uri, target uri]

    if property "filter"
        if filter starts with "_" // _view for ex
             base = base + [filter name, query_params]
        else
            fetch code of the filter function
            base = base + [code, query_params)

    repid = hex(md5(base))
</code>

Node UUID; string generated on first couchdb start and stored on the `uuid`
key of the [couchdb] section in `local.ini`. Actually can be anything that
identify the node uniquely.


### Changes of replication ID over protocols

    protocol version 2:

        base = [Host, Port, Source, Target]

    protocol version 1:

        base = [Host, Source, Target]

### Internals:

Internally the full replication Id is a tupple collecting specific options for
this replication:

    extra = []
    if continuous option:
        extra += [continuous]
    if create_target option:
        extra += [create_target]

    {replication ID, extra}


## Store a checkpoint document

A checkpoint document is a local document stored both on source and target on a
given interval. **This interval is actually fixed in source of CouchDB at
5000ms.**

The checkpoint document keeps current state of the replication and allows us
to start the replication from the last point. A checkpoint document keeps the
last 50 sessions to find the differences between the source and target.

The checkpoint document ID is `_local/replicationID` and is unique for a
replication.

### History entry properties:

- session_id
- start_time
- end_time
- start_last_seq
- end_last_seq
- recorded_seq
- missing_checked
- missing_found
- docs_read
- docs_written
- doc_write_failure

### Checkpoint document

    {
        "session_id": "string",
        "source_last_seq": "last source seq",
        "replication_id_version": 3,
        "history": [ NewHistoryEntry, ... last 49 entries ]

    }

Deprecated and only here to support the version 2 of the replication protocol,
the following properties are added:

    {
        "start_time": "ISO-8601"
        "end_time": "ISO-8601"
        "docs_read": 100
        "docs_written": 100
        "doc_write_failure": 0
    }




## CouchDB implementation

Once a replication task is posted or a replication document is created the
couch_replicator process is created. This process is responsible of keeping
the replication state updated: it receives the last source sequence updated
and update the checkpoint document. This process is also watching the source
and target to detect if they are still existing and if a compaction start.

The couch_replicator is also responsible of launching such processes:

- start changes reader process :  It adds the changes from
  the source db to the ChangesQueue.
- start changes manager: responsible for dequeing batches from the changes
  queue and deliver them to the worker processes.
- spawn replicator workers: They ask the changes queue manager for a
  a batch of _changes rows to process -> check which revs are missing in
  the target, and for the missing ones, it copies them from the source to
  the target.


For each replication tasks a pool of connection is created.


### Replication doc main properties:

- source
- target

source or target can be:

- dbname string if local
- db URI string if remote. It can contain the creadentials. Ex:

    http://<username>:<password>@<host>:<port>/<dbname>

- object:

    url: db URI string
    headers: hash of {headername, headervalue} K/V pairs
    auth:
        oauth: oauth object
            consumer key
            token
            token_secret
            consumer_secret
            signature_method (default HMAC_SHA1): HMAC_SHA1, PLAINTEXT,
                                                  RSA-SHA1


### replication options


    replication_id (when created)
    cancel
    create_target
    continuous
    filter
    query_params
    doc_ids
    worker_processes
    worker_batch_size
    http_connections
    connection_timeout
    retries_per_request
    socket_options
    since_seq
    proxy: proxy url

    _id

#### Default options:

    worker_processes = 4
    woerker_batch_size = 500
    htttp_connections = 20
    connection_timeout = 30000
    retries_per_request = 4

SSL options only set in the settings

    ssl_certificate_max_depth
    verify_ssl_certificates
    cert_file
    key_file
    password
    if ssl_trusted_certificates_file
        verify
    else
        no check
