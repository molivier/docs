=================
Object Management
=================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 1

.. container:: extlinks-video

   - `Versioning overview <https://youtu.be/XGOiwV6Cbuk?ref=docs>`__
   - `Object locking and retention overview <https://youtu.be/Hk9Z-sltUu8?ref=docs>`__
   - `MinIO Object Lifecycle Management Part I <https://youtu.be/Exg2KsfzHzI?ref=docs>`__
   - `MinIO Object Lifecycle Management Part II <https://youtu.be/5fz3rE3wjGg?ref=docs>`__

.. _objects:

An :ref:`object <objects>` is binary data, such as images, audio files, spreadsheets, or even binary executable code. 
The term "Binary Large Object" or "blob" is sometimes associated to object storage, although blobs can be anywhere from a few bytes to several terabytes in size.
Object Storage platforms like MinIO provide dedicated tools and capabilities for storing, listing, and retrieving objects using a standard S3-compatible API. 

.. include:: /includes/common-admonitions.rst
   :start-after: start-exclusive-drive-access
   :end-before: end-exclusive-drive-access

.. _buckets:

MinIO Object Storage uses :ref:`buckets <buckets>` to organize objects. 
A bucket is similar to a top-level drive, folder, or directory in a filesystem (``/mnt/data`` or ``C:\``), where each bucket can hold an arbitrary number of objects.

The structure of objects on the MinIO server might look similar to the following:

.. code-block:: text

   / #root
   /images/
      2020-01-02-MinIO-Diagram.png
      2020-01-03-MinIO-Advanced-Deployment.png
      MinIO-Logo.png
   /videos/
      2020-01-04-MinIO-Interview.mp4
   /articles/
      /john.doe/
         2020-01-02-MinIO-Object-Storage.md
         2020-01-02-MinIO-Object-Storage-comments.json
      /jane.doe/
         2020-01-03-MinIO-Advanced-Deployment.png
         2020-01-02-MinIO-Advanced-Deployment-comments.json
         2020-01-04-MinIO-Interview.md

With the example structure, an administrator would create the ``/images``, ``/videos`` and ``/articles`` buckets.
Client applications write objects to those buckets using the full "path" to that object, including all intermediate :term:`prefixes <prefix>`.

MinIO supports multiple levels of nested directories and objects using prefixes to support even the most dynamic object storage workloads.
MinIO automatically infers the intermediate prefixes, such as ``/articles/john.doe`` from the full object path using ``/`` as a delimiter. 
Clients and administrators should not create these prefixes manually.

Neither clients nor administrators would manually create the intermediate prefixes, as MinIO automatically infers them from the object name.

.. _minio-object-management-path-virtual-access:

Path vs Virtual Host Bucket Access
----------------------------------

MinIO supports both :s3-docs:`path-style <VirtualHosting.html#path-style-access>` (default) or :s3-docs:`virtual-host bucket lookups <VirtualHosting.html>`.

For example, consider a MinIO deployment with an assigned Fully Qualified Domain Name (FQDN) of ``minio.example.net``:

- With path-style lookups, applications specify the full path to a bucket, such as ``minio.example.net/mybucket``.
- With virtual-host lookups, applications specify the bucket as a subdomain, such as ``mybucket.minio.example.net/``.

Some applications may require or expect virtual-host lookup support when performing S3 operations against MinIO.
To enable virtual-host bucket lookup, you must set the :envvar:`MINIO_DOMAIN` environment variable to a :abbr:`FQDN(Fully Qualified Domain Name)` that resolves to the MinIO Deployment.

If you configure ``MINIO_DOMAIN``, you **must** consider all subdomains of the specified FQDN as exclusively assigned for use as bucket names.
Any MinIO services which conflict with those domains, such as replication targets, may exhibit unexpected or undesired behavior as a result of the collision. 

For example, if setting ``MINIO_DOMAIN=minio.example.net``, you **cannot** assign any subdomains of ``minio.example.net`` (in the form of ``*.minio.example.net``) to any MinIO service or target. 
This includes hostnames for use with :ref:`bucket <minio-bucket-replication>`, :ref:`batch <minio-batch-framework-replicate-job>`, or :ref:`site replication <minio-site-replication-overview>`.

.. important::

   For deployments with :ref:`TLS enabled <minio-tls>`, you **must** ensure your TLS certificate SANs cover all subdomains of the leftmost domain specified to :envvar:`MINIO_DOMAIN`.

   For example, the example of ``MINIO_DOMAIN=minio.example.net`` requires a TLS SAN that covers the subdomains of ``minio.example.net``.
   You can set an additional TLS SAN of ``*.minio.example.net`` to appropriately cover the subdomain namespace.

   TLS Wildcard rules prevent chaining to additional subdomain levels, such that a TLS certificate with a wildcard SAN of ``*.example.net`` would **not** cover the virtual host lookups at ``*.minio.example.net``.


Object Organization and Planning
--------------------------------

Administrators typically control the creation and configuration of buckets.
Client applications can then use :ref:`S3-compatible SDKs <minio-drivers>` to create, list, retrieve, and :ref:`delete <minio-object-delete>` objects on the MinIO deployment.
Clients therefore drive the overall hierarchy of data within a given bucket or prefix, where Administrators can exercise control using :ref:`policies <minio-policy>` to grant or deny access to an action or resource.

.. cond:: windows

   MinIO does not support the ``\`` or ``:`` characters in object names, regardless of support for those characters in Windows filesystems.
   Use ``/`` as a delimiter in object names to have MinIO automatically create a folder structure using :term:`prefixes <prefix>`.

MinIO has no hard :ref:`thresholds <minio-server-limits>` on the number of buckets, objects, or prefixes on a given deployment.
The relative performance of the hardware and networking underlying the MinIO deployment may create a practical limit to the number of objects in a given prefix or bucket.
Specifically, hardware using slower drives or network infrastructures tend to exhibit poor performance in buckets or prefixes with a flat hierarchy of objects.
For other considerations, thresholds, or limitations to keep in mind, see :ref:`minio-server-limits`.

Consider the following points as general guidance for client applications workload patterns:

- Deployments with modest or budget-focused hardware should architect their workloads to target 10,000 objects per prefix as a baseline. 
  Increase this target based on benchmarking and monitoring of real world workloads up to what the hardware can meaningfully handle. 
- Deployments with high-performance or enterprise-grade :ref:`hardware <deploy-minio-distributed-recommendations>` can typically handle prefixes with millions of objects or more.

|SUBNET| Enterprise accounts can utilize yearly architecture reviews as part of the deployment and maintenance strategy to ensure long-term performance and success of your MinIO-dependent projects.

For a deeper discussion on the benefits of limiting prefix contents, see the article on :s3-docs:`optimizing S3 performance <optimizing-performance.html>`.

Object Versioning
----------------- 

.. image:: /images/retention/minio-versioning-multiple-versions.svg
   :alt: Object with Multiple Versions
   :align: center

The specific client behavior on write, list, get, or :ref:`delete <minio-object-delete>` operations on a bucket depends on the versioning state of that bucket:

.. list-table::
   :stub-columns: 1
   :header-rows: 1
   :widths: 20 40 40
   :width: 100%

   * - Operation
     - Versioning Enabled
     - Versioning Disabled | Suspended

   * - ``PUT`` (Write)
     - Create a new full version of the object as the "latest" and assign a unique version ID
     - Create the object with overwrite on namespace match.

   * - ``GET`` (Read)
     - Retrieve the latest version of the object by default

       Supports retrieving retrieving any object version by version ID.
     - Retrieve the object

   * - ``LIST`` (Read)
     - Retrieve the latest version of objects at the specified bucket or prefix

       Supports retrieving all objects with their associated version ID.
     - Retrieve all objects at the specified bucket or prefix

   * - ``DELETE`` (Write)
     - Creates a 0-byte "Delete Marker" for the object as "latest" (soft delete)

       Supports deleting any object version by version ID (hard delete).
       You cannot undo hard-delete operations.

       Refer to :ref:`minio-object-delete` for more information.
     - Deletes the object

See :ref:`minio-bucket-versioning` for more complete documentation.

.. _minio-object-tagging:

Object Tagging
--------------

MinIO supports adding custom tags to an object.
A tag is a key-value pair included in the metadata of an object.
Tags can be used to control access with policies or locate an object with :mc-cmd:`mc find --tags`.

MinIO supports adding up to 10 custom tags to an object.

For more on setting tags, refer to :mc:`mc tag set`.

Object Retention
----------------

MinIO Object Locking ("Object Retention") enforces Write-Once Read-Many (WORM) immutability to protect :ref:`versioned objects <minio-bucket-versioning>` from deletion. 
MinIO supports both :ref:`duration based object retention <minio-object-locking-retention-modes>` and :ref:`indefinite legal hold retention <minio-object-locking-legalhold>`.

.. image:: /images/retention/minio-object-locking.svg
   :alt: 30 Day Locked Objects
   :align: center
   :width: 600px

Delete operations against a WORM-locked object depend on the specific operation:

- Delete operations which do not specify a version ID result in the creation of a "Delete Marker"
- Delete operations which specify the version ID of a locked object result in a WORM locking error

You can only enable object locking when first creating a bucket.
Enabling bucket locking also enables :ref:`versioning <minio-bucket-versioning>`.

MinIO Object Locking provides key data retention compliance and meets SEC17a-4(f), FINRA 4511(C), and CFTC 1.31(c)-(d) requirements as per `Cohasset Associates <https://min.io/cohasset?ref=docs>`__.

See :ref:`minio-object-locking` and :ref:`minio-object-delete` for more complete documentation.

Object Lifecycle Management
---------------------------

MinIO Object Lifecycle Management allows creating rules for time or date based automatic transition or expiry of objects. 
For object transition, MinIO automatically moves the object to a configured remote storage tier. 
For object expiry, MinIO automatically deletes the object.

MinIO applies lifecycle management rules on :ref:`versioned and unversioned buckets <minio-bucket-versioning>` using the same behavior as normal client operations.
You can specify transition or lifecycle rules that handle the latest object versions, non-current object versions, or both.

MinIO lifecycle management is built for behavior and syntax compatibility with :s3-docs:`AWS S3 Lifecycle Management <object-lifecycle-mgmt.html>`. 
MinIO uses JSON to describe lifecycle management rules.
Conversion to or from XML may be required for importing rules created on S3 or similar compatible platforms. 

See :ref:`minio-lifecycle-management` for more complete documentation.

Target Bucket Considerations
----------------------------

MinIO does *not* require that the target bucket match object management or versioning configurations with the source bucket.
The target bucket *may* have its own set of object management rules, if defined with care.

Target buckets should *not* have their own rules for expiration or additional tiering.
Expiration rules can result in removal of tiered data still in use by the source bucket.
Tiering to an additional remote creates an additional network hop between the hot tier and it's data while also increasing operational complexity.

You *may* configure object locking or versioning on the remote bucket.

Enabling versioning or object locking on the target bucket may have effects such as the following:

- Object locking set on the target bucket may prevent desired ``delete`` operations from the source bucket from completing.
- MinIO tiers objects with their own ``UUID``, so versioning on the target bucket is redundant at best.
- Reduced storage efficiency on the target, as ``delete`` operations result in creation of a ``DeleteMarker`` rather than freeing space.
- Duplicate delete markers on source and target buckets.

Exclusive Access to Remote Data
-------------------------------

MinIO **must** have *exclusive* access to the target bucket.
No other user, process, application, or resource should have any access to or perform any actions against the target bucket.

All access to the transitioned objects *must* occur through MinIO via S3 API operations only. 
Manually modifying a transitioned object - whether the metadata on the “hot” MinIO tier or the object data on the remote “warm/cold” tier - may result in loss of that object data.

MinIO ignores any objects in the remote bucket or bucket prefix not explicitly managed by the MinIO deployment. Automatic transition and transparent object retrieval depend on the following assumptions:

- No external mutation, migration, or deletion of objects on the remote storage.
- No lifecycle management rules (such as transition or expiration) on the remote storage bucket.

To facilitate this exclusive access, grant the lifecycle management user ``read``, ``write``, and ``delete`` access to the target bucket in its :ref:`policy <minio-policy>`.
All other policies should ``deny`` access to the target bucket.


Conflicting Objects
-------------------

Applications must assign non-conflicting, unique keys for all objects.
This includes avoiding creating objects where the name can collide with that of a parent or sibling object.
MinIO returns an empty set for LIST operations at the location of the collision.

For example, the following operations create a namespace conflicts

.. code-block::
   
   PUT data/invoices/2024/january/vendors.csv
   PUT data/invoices/2024/january <- collides with existing object prefix

.. code-block::

   PUT data/invoices/2024/january
   PUT data/invoices/2024/january/vendors.csv <- collides with existing object

While you can perform GET or HEAD operations against these objects, the name collision causes LIST operations to return an empty result set at the ``/invoices/2024/january`` path.


.. toctree::
   :titlesonly:
   :hidden:

   /administration/object-management/object-versioning
   /administration/object-management/object-retention
   /administration/object-management/object-delete
   /administration/object-management/object-lifecycle-management
   /administration/object-management/data-compression
