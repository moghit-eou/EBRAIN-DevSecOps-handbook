# Why three independent pipelines

TODO: why Container Scanning, SCA, and SAST gate independently instead of
as one combined job. Cover why Container Scanning itself runs two scan
types (sast against the Dockerfile, sca against the built image) rather
than being folded entirely into the other two pipelines.
