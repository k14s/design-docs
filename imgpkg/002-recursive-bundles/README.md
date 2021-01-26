---
tags: recursive bundles, imgpkg
---
# imgpkg Recursive Bundles Proposal

- Proposal Status: **Draft**, In Review, Accepted, Rejected


# Table of Contents

[TOC]

---

# Recursive Bundles Proposal

## Goals

- Propose changes in configuration to support bundles pointing to bundles
- Provide 

## Anti Goals

- Prescribe implementation details


# Scenarios

**As** A Software Packager
**I want to** Create a bundle to distribute the software
**So that** It can be used by my users

In a scenario where a Packager is creating a package that contain multiple applications, there should be a way to create a single artifact that could be used by the Users to install these applications.

Lets assume the following scenario, with 2 applications that need to be packaged together.

Contents of each application

    Application 1:
      - Application Image
      - Configuration

    Application 2:
      - Application Image
      - Configuration

The developer of `Application 1` creates a bundle with the application and the needed configuration.

The developer of `Application 2` creates a bundle with the application and the needed configuration.

The packager will creates a bundle that uses both bundles for `Application 1` and `Application 2` plus any extra configuration needed

#### Application 1
##### Bundle definition
.imgpkg/images.yml:
```
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: ImagesLock
images:
- image: some.registry.io/application1@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0
```

##### Creation of the bundle
Executing `imgpkg push -b some.registry.io/app1-bundle -f .` creates the bundle in `some.registry.io/app1-bundle`

#### Application 2
##### Bundle definition
.imgpkg/images.yml:
```
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: ImagesLock
images:
- image: some.registry.io/application1@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0
```

##### Creation of the bundle
Executing `imgpkg push -b some.registry.io/app1-bundle -f .` creates the bundle in `some.registry.io/app1-bundle`



# Impacts
Some impacts in the current implementation:
- Assumption that bundles cannot contain bundles


# Considerations
## Creating bundle

When creating bundles no changes should be made. This proposal will only allow creation of bundles, from a folder that contains only 1 .imgpkg folder.

## Copying bundle
### To a tarfile

Major changes:
- remove the constraint that does not allow bundles to contain other bundles
- Navigate throught the bundles and downloading each layer


Need to ensure that layers with same SHA only are present once inside the tar.

Tar structure should still be the same:
```bash
$ tree
.
├── manifest.json
├── sha256-26bef58ee14200d194347468d91506440cbee2c69622036d057ca862948451bd.tar.gz
└── sha256-8ece9ac45f2b7228b2ed95e9f407b4f0dc2ac74f93c62ff1156f24c53042ba54.tar.gz
```

### From tarfile to repository

No changes

### From repository to repository

Major changes:
- remove the constraint that does not allow bundles to contain other bundles
- Navigate throught the bundles and copy the images

## Pull Bundle
### Simple bundles

No changes

### Bundles of bundles

Major changes:
- When retrieving the bundle recursively iterate over the image lock
- Download the bundle information in particular folders

Folder structure for pulled bundles:
```
$ tree -a ../examples/basic-step-2
.
├── .bundles
│   ├── sha256-26bef58ee14200d194347468d91506440cbee2c69622036d057ca862948451bd
│   │   ├── .imgpkg
│   │   │   ├── bundle.yml
│   │   │   └── images.yml
│   │   └── config2.yml
│   └── sha256-8ece9ac45f2b7228b2ed95e9f407b4f0dc2ac74f93c62ff1156f24c53042ba54
│   │   ├── .imgpkg
│   │   │   ├── bundle.yml
│   │   │   └── images.yml
│       └── config1.yml
├── .imgpkg
│   ├── bundle.yml
│   └── images.yml
└── config.yml

7 directories, 9 files
```

---
---
Random notes:

historical:
- no a lot of conversations
- we knew we would need them

Maybe an aliviated way to get the bundle moved
we already check if it is a bundle or not

gist from Tim
unsure

intent to have a kapp-controller annotation to

create 

collisions between images
- 2 bundles that reference the same bundle is ok we should not upload it twice
- need to ensure we do not have cyclic bundles

Consider creating a blob with the manifest instead of storing them in the Manifest.json that we save in the tar



Eventually filter relocated images so that we do not create these images

Maybe it is interesting to get the list of images that are being bundled

How to pull bundle of bundles into disk? What happens? where do we put the config layers

