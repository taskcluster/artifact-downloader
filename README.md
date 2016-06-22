# Artifact Downloader
The taskcluster platform stores a lot of things as artifacts.  We have to
download these artifacts in a number of different places.  Right now, each
downloader of artifacts has to implement its own downloading logic.  This means
that we have multiple tools following redirects, doing retries and each is a
sub-optimal implementation.  It also means that in projects that are run by
taskcluster-github and taskcluster-mercurial need to understand curl or wget.

The artifact downloader program will be the official way to download artifacts
from the Taskcluster infrastructure.  The main features we'd like to implement
are:

1. simple addressing of artifacts.  No URL generation needed to download a file
   but able to use generated urls
1. retry logic with exponential backoff that's the same algorithm with useful
   defaults
1. useful debugging messages when there's a failure to download
1. ability to verify common file types are actually what is downloaded
1. simple deployment as a single binary
1. force tool to only follow ssl-protected resources
1. ability to set min and max expected sizes as a sanity check
1. if the queue provides a cache clearing endpoint, ability to automatically
   purge caches on content errors
1. not subject to silly things like bug 1276110
1. can be exposed in taskcluster-github worker path to simplify
   taskcluster-github implementation

# Debugging output 
One of the most common situations right now when something goes wrong is that
we have not way of determining what when wrong during the download.  The tool
by default will print out a set of important fields, like http status code,
Location header for redirects, content-type, content-md5, content-encoding and
other critical fields.  It will optionally print out all headers from each
redirect.

When artifact validation is used and the artifact is either too large, too
small or not in the expected format, the first 1024 bytes of the document will
be printed to stdout to make debugging easier.  This is especially helpful
when, for instance S3 error pages are downloaded in place of a .tar.gz file.

# Usage
Note that this was written before the tool was implemented.  This is subject to
change.

The base command is named `artifact-download` and can take the following
options:

* `--url=$URL`: used when a fully formed artifact url is already known.
  Incompatible with `--taskid` and `--name`
* `--taskid=$TASKID`: used when an artifact url needs to be determined,
  requires `--name`
* `--name=$ARTIFACT_NAME`: specify the name of the artifact requested, requires
  `--taskid`
* `--queue=$QUEUE_HOSTNAME`: (optional) specify an alternate location of
  `queue.taskcluster.net`, compatible only with --taskid and --name. Example:
  `http://taskcluster/queue`.  Alternately set with environment variable
  `TASKCLUSTER_QUEUE_BASE`
* `--format=[tar-gz|tar-xz|exe|xml|json|etc...]`: (optional) specifies the
  format expected and implies that output is to be validated.
* `--extract=$DESTINATION`: (optional) if format is a known archive format,
  extract to the specified location, or present working directory if
  unspecified
* `--retries=$NUM_RETRIES`: (optional) if specified, limit max number of
  retries to this number
* `--purge-on-error`: (optional) if specified, on error try to purge cache that
  the queue uses
* `--debug`: (optional) if specified, be extra verbose, overrides `--silent`
* `--silent`: (optional) if specified, be as quiet as possible
* `--size=`: (optional) if specified, a string in the format `x:y` where x and
  y are both integers showing the number of bytes expected.  The y value must
  be greater than the x value.  Both x and y must be number of bytes.  This
  will be validated against the length of the actual file, not Content-Length.
  A y value of 0 is interpreted as infinitely large.  This value might be used
  to terminate download attempts

# Example Usage

### Download an artifact using a known URL
    artifact-download \
      --url=https://queue.taskcluster.net/v1/task/N39CRjUXQ7aq2OEexeJphg/artifacts/public/build/target.tar.gz \
      --format=tar-gz \
      --extract=workspace/build \
      --purge-on-error \
      --size=$((10 * 1024 * 1024)):$((500 * 1024 * 1024))

### Download an artifact using the taskcluster proxy
Note that in production, the TASKCLUSTER_QUEUE_URL variable would be set by the
docker worker

    TASKCLUSTER_QUEUE_URL=http://taskcluster/queue/ artifact-download \
      --taskid=N39CRjUXQ7aq2OEexeJphg \
      --name=public/build/target.tar.gz \
      --format=tar-gz \
      --extract=workspace/build \
      --purge-on-error \
      --size=$((10 * 1024 * 1024)):$((500 * 1024 * 1024))

