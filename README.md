# Implementation of Disaster Recovery For NFS

The Distributed Replicated Block Device (DRBD), is a data storage that uses more than one servers. A “primary” server stores the data, and the other “secondary” server that replicate the contents of the primary. If the primary server fails, one of the secondary servers will then become the primary, to prevent data loss.
In this project two NFS server will server this purpose.




