---
tags: recursive bundles, imgpkg
---
# imgpkg Recursive Bundles Proposal

- Proposal Status: **Draft**, In Review, Accepted, Rejected


# Table of Contents

[TOC]

---

# Recursive Bundles Proposal

## Scenarios

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



# Proposed changes

In this section the proposal will only talk about parts that will changed based on the Recursive Bundle solution

## Pushing a bundle

The major proposed change in this section is to remove the validation done today that checks if the bundle being pushed contains bundles in the `.imgpkg/images.yml` Lock file.

### Required changes

#### Validation removal

Today we validate to ensure that when we are pushing a bundle, the images associated with it are not bundles.
The error message that is shown read `Error: Expected image lock to not contain bundle reference:`

As part of this proposal this check no longer will be done.

#### Ignore bundle specific folder

When a bundle is pulled, and contain recursive bundles, the folder `.bundles` is created in the root of the bundle.
As part of this proposal we should ensure that if a user tries create a bundle and it contains a folder called `.bundles`
`imgpkg` should ignore the folder and provide the following message

```
Warning: .bundles folder could not be added to the bundle.
         This folder is used to store configurations of recursive bundles and cannot be checked into the bundle
```

### Example

Assuming the provided example in [here](https://github.com/k14s/design-docs/imgpkg/002-recursive-bundles/examples)

```=
$ imgpkg push -b my.registry.io/application-bundle -f imgpkg/002-recursive-bundles/examples/bundle-1
dir: .
file: .imgpkg/bundle.yml
file: .imgpkg/images.yml
file: config.yml
file: Readme.md
Pushed 'my.registry.io/application-bundle@sha256:24a39b8597aace43b4361110c43d6548a350380dcd265fb896d29701108c61c7'
Succeeded
```

## Copy a bundle

### From Repository to Tar

#### Required changes

##### Retrieve all images for all bundles

The most significant change in this operation is that when we copy the images `imgpkg` will have to traverse all the bundles to collect all the images to copy.

##### Layer deduped on disk 

When creating the tar file in disk `imgpkg` need to ensure that we do not store duplicated layers in disk

#### Example

Given that we create a bundle using the command 

`imgpkg push -b my.registry.io/recursive-bundle -f imgpkg/002-recursive-bundles/examples/recursive-bundle`

```=
imgpkg copy -b my.registry.io/recursive-bundle --to-tar recursive-bundle.tar
copy | exporting 4 images...
copy | will export my.registry.io/simple-application@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0
copy | will export my.registry.io/utilities@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b
copy | will export my.registry.io/application-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5
copy | will export my.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e
copy | exported 4 images
copy | writing layers...
copy | done: file 'manifest.json' (35.46µs)
copy | done: file 'sha256-4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0.tar.gz' (67.001µs)
copy | done: file 'sha256-0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b.tar.gz'
copy | done: file 'sha256-65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5.tar.gz' (67.001µs)
copy | done: file 'sha256-9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e.tar.gz'  (428.476213ms)
Succeeded
```

### From Tar to Repository

#### Required changes

##### Retrieve all images for all bundles

The most significant change in this operation is that when we copy the images `imgpkg` will have to traverse all the bundles to collect all the images to copy.

#### Example

Given that we create a bundle using the command `imgpkg push -b my.registry.io/recursive-bundle -f imgpkg/002-recursive-bundles/examples/recursive-bundle`

Followed by
`imgpkg copy -b my.registry.io/recursive-bundle --to-tar recursive-bundle.tar`

```=
imgpkg copy --tar recursive-bundle.tar --to-repo some-other.registry.io/recursive-bundle
copy | importing 4 images...
copy | importing my.registry.io/simple-application@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0 -> some-other.registry.io/recursive-bundle@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0...
copy | importing my.registry.io/utilities@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b -> some-other.registry.io/recursive-bundle@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b...
copy | importing my.registry.io/application-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5 -> some-other.registry.io/recursive-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5...
copy | importing my.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e -> some-other.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e...
copy | imported 4 images
Succeeded
```

### From Repository to Repository

#### Required changes

##### Retrieve all images for all bundles

The most significant change in this operation is that when we copy the images `imgpkg` will have to traverse all the bundles to collect all the images to copy.

##### Bundles/Images deduped

When copying the images/bundles ensure that `imgpkg` does not try to copy the same image multiple times

#### Example

Given that we create a bundle using the command 

`imgpkg push -b my.registry.io/recursive-bundle -f imgpkg/002-recursive-bundles/examples/recursive-bundle`

```=
imgpkg copy -b my.registry.io/recursive-bundle --to-repo some-other.registry.io/recursive-bundle
copy | exporting 4 images...
copy | will export my.registry.io/simple-application@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0
copy | will export my.registry.io/utilities@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b
copy | will export my.registry.io/application-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5
copy | will export my.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e
copy | exported 4 images
copy | importing 4 images...
copy | importing my.registry.io/simple-application@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0 -> some-other.registry.io/recursive-bundle@sha256:4c8b96d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e63a9d0...
copy | importing my.registry.io/utilities@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b -> some-other.registry.io/recursive-bundle@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b...
copy | importing my.registry.io/application-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5 -> some-other.registry.io/recursive-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5...
copy | importing my.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e -> some-other.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e...
copy | imported 4 images
Succeeded
```

## Pull a bundle

### Required changes

#### Change folder structure

When downloading a bundle that contains other bundles to disk using the `pull` command will work as currentlly, except for that it will download the nested bundles into a hidden folder called `.bundles`.

##### Folder structure

The structure of the output folder will be
```
$ tree -a recursive-bundle
.
├── .bundles
│   ├── sha256-{SHA Of the First Nested Bundle}
│   │   ├── .imgpkg
│   │   │   ├── bundle.yml
│   │   │   └── images.yml
│   │   └── config2.yml
│   └── sha256-{SHA Of the Second Nested Bundle}
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

Folder name will be the sha256-{SHA} where SHA is the SHA256 of the bundle OCI Image. The mapping between the folder names and the origin images can be found in the `.imgpkg/images.yml` of the bundle that included this bundle

###### Cyclic nesting

To ensure that there is not cyclic nesting in disk `imgpkg` will flatten the bundle structure to 1 level.
![](https://i.imgur.com/zSlnzg7.png)
In the image above Bundle 1, Bundle 2 and, Bundle 3 will be all in a single level inside `.bundles` folder.


### Example

Given that we create a bundle using the command `imgpkg push -b my.registry.io/recursive-bundle -f imgpkg/002-recursive-bundles/examples/recursive-bundle`

```=
$ imgpkg pull -b my.registry.io/recursive-bundle -o /tmp/recursive-bundle

Pulling bundle 'some-other.registry.io/recursive-bundle@sha256:9abdbad2e7953a64d54407ddb05241560f07199561113896dfea4990b700680e'
Bundle Layers
  Extracting layer 'sha256:83025c5ef551634daf7daa2f83ded5e7a418b12c66d66622940b3f3c60f53f05' (1/1)
  
Nested bundles
  Pulling Bundle 'some-other.registry.io/recursive-bundle@sha256:0270af71de717e3bd709c8df25462f0fbfa7c2c92b730aa58314ee83253e6e0b' (1/2)
  Extracting layer 'sha256:aaaaa6d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e6aaaaa' (1/1)
    Found 1 Bundle packaged

    Pulling Nested Bundle 'some-other.registry.io/recursive-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5' (1/1)
    Extracting layer 'sha256:bbbbb6d4fffdfae29258d94a22ae4ad1fe36139d47288b8960d9958d1e6bbbbb' (1/1)
  
  Pulling Nested Bundle 'some-other.registry.io/recursive-bundle@sha256:65eb68ccf87929587c7434cdf57664378008184f45abc19dd680159b536f53c5' (2/2)
  Skipped, already downloaded

Locating image lock file images...
The bundle repo (some-other.registry.io/recursive-bundle) is hosting every image specified in the bundle's Images Lock file (.imgpkg/images.yml)

Updating all images in the ImagesLock file: pull-tmp/.imgpkg/images.yml
+ Changing all image registry/repository references in pull-tmp/.imgpkg/images.yml to some-other.registry.io/recursive-bundle/test-2
```


---
---
:construction: Below this point we only have notes :construction: 
---
---


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

