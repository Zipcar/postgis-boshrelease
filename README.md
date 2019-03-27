# postgis-boshrelease
---

## Contents

* [Summary](#summary)
* [Supported Postgres Versions](#supported-postgres-versions)
* [Deploying](#deploying)
* [Contributing](#contributing)
* [Considerations before deploying](#considerations-before-deploying)
* [Considerations after a successful deployment](#considerations-after-a-successful-deployment)

## Summary 

This is a [BOSH](https://www.bosh.io) release for [PostGis](https://postgis.net), which builds on top of the 
[Postgres release](https://github.com/cloudfoundry/postgres-release).

It provides a set of modules and extensions that are loaded into postgres at runtime, and thereby make various
spatial functions available to postgres users.

Specifically, it adds the following postgres extensions:

  - postgis
  - postgis_topology
  - postgis_sfcgal
  - fuzzystrmatch
  - address_standardizer
  - address_standardizer_data_us
  - postgis_tiger_geocoder

## Supported Postgres Versions 

This release supports the following postgres versions:

 * postgres 9.6
 * postgres 10.7
 * postgres 11.1
 * postgres 11.2

Only one postgres version can be active at a time.

It is currently tested against the following postgres-releases:

 * [v31](https://github.com/cloudfoundry/postgres-release/releases/tag/v31) - provides postgres 9.6
 * [v36](https://github.com/cloudfoundry/postgres-release/releases/tag/v36) - provides postgres 11.2

It provides the following dependencies:

 * [boost](https://www.boost.org/) 1.67.0
 * [CGAL](https://www.cgal.org/) 4.13
 * [gdal](https://www.gdal.org/) 2.4.0
 * [geos](https://trac.osgeo.org/geos/) 3.7.1
 * [gmp](https://gmplib.org/) 6.1.2
 * [json-c](http://json-c.github.io/json-c/) 0.13.1
 * [libxml2](http://xmlsoft.org/) 2.9.9
 * [mpfr](https://www.mpfr.org/) 4.0.2
 * [proj](https://proj4.org) 5.2.0
   * [datumgrid](https://github.com/OSGeo/proj-datumgrid) 1.8
   * [sqlite](https://www.sqlite.org) 3.27.2
 * [pcre](https://www.pcre.org/) 8.42
 * [SFCGAL](http://oslandia.github.io/SFCGAL) 1.3.6

## Deploying

This release has the same pre-requisites as the [postgres release](https://github.com/cloudfoundry/postgres-release).
You will need a working bosh director and you must have a way to upload releases to it (either development releases
or final releases). You should generate a deployment manifest as per the postgres release's documentation.

The following instructions assume you have a valid deployment manifest that will deploy an instance of the postgres
release. You may have generated this using the `generate-deployment-manifest` script provided by the postgres-release.

1. Add the postgis-release to your deployment manifest

In the `releases` block, add a reference to the `postgis-release`, as follows:

  ```
  releases:
  - name: postgis
    version: latest
  ```
(be sure to leave the `postgres` release reference intact).

1. Add the postgis add-on to your manifest

The postgis job should be included as an `addon`. Place the following at the end of your deployment manifest:

  ```
  addons:  
  - name: postgis
    jobs:
    - name: postgis
      release: postgis
  ``` 
   
1. Deploy:

   ```
   bosh -d DEPLOYMENT_NAME deploy OUTPUT_MANIFEST_PATH
   ```

## Contributing

This bosh release is open source. PRs/issues/etc welcome.

Contributions that track postgis dependencies (e.g. gdal, postgres-release) most welcome.

## Considerations before deploying

Make sure to back up any existing postgres data before deploying. This release will not alter any of your existing
data but does alter the behaviour of postgres itself.

## Considerations after a successful deployment

A number of new database tables and objects are available after a successful deployment. To access those you should
execute the following queries as a db user with sufficient permissions (eg `postgres`).

### Creating new extensions

The following extensions are provided by this boshrelease. To enable them run some subset of the following queries
as a db user with sufficient permissions (eg `postgres`):

  ```
  CREATE EXTENSION IF NOT EXISTS postgis
  CREATE EXTENSION IF NOT EXISTS postgis_topology
  CREATE EXTENSION IF NOT EXISTS postgis_sfcgal
  CREATE EXTENSION IF NOT EXISTS fuzzystrmatch
  CREATE EXTENSION IF NOT EXISTS address_standardizer
  CREATE EXTENSION IF NOT EXISTS address_standardizer_data_us
  CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder

  ```

### Allowing read access to new spatial tables

You may encounter an error like this:

  ```
  spatial_ref_system relation does not exist
  ```

This may be because the db user does not have access to the `spatial_ref_sys` table that postgis provides. You
can grant access as follows (note that this also grants access to several other tables that your users probably
need).

Allowing all users in your database to read the spatial tables:

  ```
  GRANT SELECT ON TABLE public.geometry_columns TO PUBLIC
  GRANT SELECT ON TABLE public.spatial_ref_sys TO PUBLIC
  GRANT SELECT ON TABLE public.raster_columns TO PUBLIC
  GRANT SELECT ON TABLE public.raster_overviews TO PUBLIC
  ```

You may prefer a more restricted model whereby you only grant access to particular postgres users, or sets of users
in specific schemas. In that case something like this might be preferred:

  ```
  --- using a particular db schema...
  GRANT SELECT ON TABLE geometry_columns TO <your_user>
  GRANT SELECT ON TABLE spatial_ref_sys TO <your_user>
  GRANT SELECT ON TABLE raster_columns TO <your_user>
  GRANT SELECT ON TABLE raster_overviews TO <your_user>
  ```

A future version of the postgis-release would automate some of these steps. For now we leave this as a choice for
the operator.
