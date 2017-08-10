Automatic Volume Snapshots on Kubernetes / Google Container Engine
==================================================================

1. You add an annotation to your Kubernetes `PersistentVolume` resource,
   to indicate how many backups should be kept.
2. This tool will create and expire snapshots to match the desired
   configuration.

Currently, only the following setup is supported:

- Google Compute Engine disks.


How it works
------------

1. Run it as a service, preferable inside your Kubernetes cluster.
2. It will watch `PersistentVolume` resources and check those for an
   annotation named `backup.kubernetes.io/deltas`, which would have
   a value such as `1d 7d 30d`.
3. For every `PersistentVolume` that defines this annotation, and is
   a Google Compute disk, it will create new snapshots, and delete
   existing snapshots, according to the deltas defined.

  **WARNING**: The tool *will* consider snapshots not created by it.
  It will consider, and potentially delete, every snapshot that is
  associated with the disk in question.


How do the deltas work
----------------------

The expiry logic of [tarsnapper](https://github.com/miracle2k/tarsnapper)
is used.

The generations are defined by a list of deltas formatted as [ISO 8601
durations](https://en.wikipedia.org/wiki/ISO_8601#Durations) (this differs from
tarsnapper). ``PT60S`` means a minute, ``PT12H`` is half a day, ``PT7D`` is a
week. The number of backups in each generation is implied by it's and the parent
generation's delta.

For example, given the deltas ``PT1H P1D P7D``, the first generation will
consist of 24 backups each one hour older than the previous
(or the closest approximation possible given the available backups),
the second generation of 7 backups each one day older than the previous,
and backups older than 7 days will be discarded for good.

The most recent backup is always kept.

The first delta is the backup interval.


Usage
-----

Run it with docker:

    docker run -e ... elsdoerfer/k8s-snapshots


Add annotations to your PersistentVolumes. If those volumes are auto
generated by a provisioner based on a PersistentVolumeClaim, you cannot
currently (it seems to me) define inside your claim which annotations
the volume should have. To enable backups for a volume, add the
deltas annotation manually:

    $ kubectl edit pv pvc-afee65c7-d014-084a-b158-42010af000bd

Add an annotation such as:

    backup.kubernetes.io/deltas: PT1H P2D P30D P180D


Configuration
-------------

Provide environment variables to configure these.

<table>
  <tr>
    <th>Variable name</th>
    <th>Required</th>
    <th>Default</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>GCLOUD_PROJECT</td>
    <td>Yes</td>
    <td></td>
    <td>Name of the Google Cloud project</td>
  </tr>
  <tr>
    <td>GCLOUD_JSON_KEYFILE_NAME</td>
    <td>One GCloud auth method is required</td>
    <td></td>
    <td>
      Filename to the JSON keyfile that is used to authenticate. You'll want
      to mount it into the container.
    </td>
  </tr>
  <tr>
    <td>GCLOUD_JSON_KEYFILE_STRING</td>
    <td>One GCloud auth method is required</td>
    <td></td>
    <td>
      The contents of the JSON keyfile that is used to authenticate.
    </td>
  </tr>
  <tr>
    <td>KUBE_CONFIG_FILE</td>
    <td>No</td>
    <td>Automatically uses the service account associated with the pod.</td>
    <td>
      Authentification with the Kubernetes API.
    </td>
  </tr>
  <tr>
    <td>USE_CLAIM_NAME</td>
    <td>No</td>
    <td>False</td>
    <td>If set, and the name of the volume is known to be autogenerated
        by the provisioner, and the volume is bound to a claim,
        then use the namespace/name of the claim as the name for the
        snapshots.
    </td>
  </tr>
  <tr>
    <td>LOG_LEVEL</td>
    <td>No</td>
    <td>INFO</td>
    <td>DEBUG, INFO, WARNING, ERROR</td>
  </tr>
  <tr>
    <td>VOLUMES</td>
    <td>No</td>
    <td></td>
    <td>
      Comma-separated list of volumes to backup. This allows you to
      manually specify volumes you want to create snapshots for; useful
      for volumes you are using without a PersistentVolume.
    </td>
  </tr>
  <tr>
    <td>VOLUME_{NAME}_DELTAS</td>
    <td>Yes</td>
    <td></td>
    <td>
      The deltas for this volume.
    </td>
  </tr>
  <tr>
    <td>VOLUME_{NAME}_ZONE</td>
    <td>Yes</td>
    <td></td>
    <td>
      The zone for this volume.
    </td>
  </tr>
  <tr>
    <td>PING_URL</td>
    <td>No</td>
    <td></td>
    <td>
      We'll send a GET request to this url whenever a backup completes.
      This is useful for integrating with monitoring services like
      Cronitor or Dead Man's Snitch.
    </td>
  </tr>
</table>


Development
-----------

For local development, you can still connect to an existing Google
Cloud Project and Kubernetes cluster using the config options
available. If you are lucky, your local workstation is already setup
the way you need it. If we can find credentials for Google Cloud
or Kubernetes, they will be used automatically. If so, the following
command should work for you:

    $ GCLOUD_PROJECT=handy-hexagon python -m k8s_snapshots
