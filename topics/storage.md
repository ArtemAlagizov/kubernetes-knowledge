## storage
* persistent volume => cluster wide volume
  * default persistentVolumeReclaimPolicy: Retain
* persistent volume claim => request for a persistent volume with some criteria (min size, ...)
* map regular volume to hostpath
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: webapp
  spec:
    containers:
    - name: event-simulator
      image: kodekloud/event-simulator
      env:
      - name: LOG_HANDLERS
        value: file
      volumeMounts:
      - mountPath: /log
        name: log-volume

    volumes:
    - name: log-volume
      hostPath:
        # directory location on host
        path: /var/log/webapp
        # this field is optional
        type: Directory
  ```
* persistent volume for logs
  ```
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-log
  spec:
    accessModes:
      - ReadWriteMany
    capacity:
      storage: 100Mi
    hostPath:
      path: /pv/log
  ```
* persistent volume claim for this directory
  ```
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: claim-log-1
  spec:
    accessModes:
    - ReadWriteMany
    resources:
      requests:
        storage: 50Mi
  ```
* configure pod to use pvc
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: webapp
  spec:
    containers:
    - name: event-simulator
      image: kodekloud/event-simulator
      env:
      - name: LOG_HANDLERS
        value: file
      volumeMounts:
      - mountPath: /log
        name: log-volume

    volumes:
    - name: log-volume
      persistentVolumeClaim:
        claimName: claim-log-1
  ```
* aws ebs example config
  ```
  ...
  volumes:
  - name: data-volume
    awsElasticBlockStorage:
      volumeId: <volumeId>
      fsType: ext4
  ```
