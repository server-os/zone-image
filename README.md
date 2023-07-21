# Server OS Zone Image

Build the image used to create Server OS zones.

## Prerequisites

- Server OS based build server is available with server-image and zone-image repos
- The Server Image has been built (see https://github.com/server-os/server-image)

Note: The zone image has to be built from within the global zone!

## Create seed image

In the global zone, navigate to the zone-image repo on the build server zone:

    $ cd /zones/<build-server-uuid>/root/root/zone-image

And create the seed image:

    $ ./create-seed ../server-image zone-image-seed

Install the seed image:

    $ imgadm install -m ./output/zone-image-seed.json -f ./output/zone-image-seed.zfs.gz

Cleanup the seed image:

    $ zfs destroy zones/zone-image-seed








imagetools
==========

Tools for creating SmartOS images.

## Creating seed images

Seed images are the absolute baseline of a SmartOS image.  They are created
from files produced by a smartos-live build, containing the original `/etc`,
`/var` and SMF manifest database.

First, you need to perform a smartos-live build, as described in the wiki
page:

  http://wiki.smartos.org/display/DOC/Building+SmartOS+on+SmartOS

For this example we are building the `release-20141030` tag, in order to
produce a `seed-20141030` image off that baseline.

Then from the global zone:

    $ ./create-seed /path/to/smartos-live seed-20141030

This will create a `zones/seed-20141030` file system containing the seed files.

Once you have a seed file system ready, use `create-image` to turn it into a
provisionable image and manifest.

    $ ./create-image seed-20141030 seed-20141030

This will use `/zones/seed-20141030`, snapshot it, and create a file system
image along with a manifest file ready for importing with `imgadm`.

Once complete it will output an `imgadm` command you can use, e.g.

    Done.  Now run this to install the image:
    
      imgadm install -m ./output/seed-20141030.json -f ./output/seed-20141030.zfs.gz

If that's successful you can cleanup the temporary dataset:

    $ zfs destroy zones/seed-20141030

You can then use the UUID as input for the next phase.

## Creating base images

Base images are comprised of a baseline seed image, plus an applied overlay
appropriate for the target image.  You use the `install-image` tool to copy
the overlay files into a specified zone and then execute the customize script.

Start by creating a basic zone based on the seed image created above (replacing
`image_uuid` with the uuid of the seed image):

    $ vmadm create <<EOF
    {
      "brand": "joyent",
      "image_uuid": "1e9e46ec-e4e5-11e4-9bdb-1788911817ce",
      "max_physical_memory": 512,
      "alias": "seed-zone",
      "nics": [
        {
          "nic_tag": "admin",
          "ip": "dhcp"
        }
      ]
    }
    EOF
    Successfully created VM c374c4bc-2395-4848-b28d-0c18937e7775

Then we can apply the `2014Q4-i386` configuration to the VM with:

    $ ./install-base -c 2014Q4-i386 -n base-32-lts -r 14.4.0 -z c374c4bc-2395-4848-b28d-0c18937e7775

This uses the `2014Q4-i386` configuration, sets the version number to `14.4.0`
and installs to the specified zone.

The final part of this script runs `sm-prepare-image` which does some final
image cleanup and shutdown, after which you can simply generate the finished
image, again with `create-image`:

    $ ./create-image base-32-lts-14.4.0 c374c4bc-2395-4848-b28d-0c18937e7775

This final image and manifest should now be suitable for production use.
