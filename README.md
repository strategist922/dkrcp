# dkrcp
Copy files between host's file system, containers, and images.
#####ToC
[Copy Semantics](#copy-semantics)  
&nbsp;&nbsp;&nbsp;&nbsp;[Images as SOURCE/TARGET](images-as-sourcetarget)  
[Install](#install)

```
Usage: ./dkrcp.sh [OPTIONS] SOURCE [SOURCE]... TARGET 

  SOURCE - Can be either: 
             host file path     : {<relativepath>|<absolutePath>}
             image file path    : [<nameSpace>/...]{<repositoryName>:[<tagName>]|<UUID>}::{<relativepath>|<absolutePath>}
             container file path: {<containerName>|<UUID>}:{<relativepath>|<absolutePath>}
             stream             : -
  TARGET - See SOURCE.

  Copy SOURCE to TARGET.  SOURCE or TARGET must refer to either a container/image.
  <relativepath> within the context of a container/image is relative to
  container's/image's '/' (root).

OPTIONS:
    --ucpchk-reg=false        Don't pull images from registry. Limits image name
                                resolution to Docker local repository for  
                                both SOURCE and TARGET names.
    --author="",-a            Specify maintainer when TARGET is an image.
    --change[],-c             Apply specified Dockerfile instruction(s) when
                                TARGET is an image. see 'docker commit'
    --message="",-m           Apply commit message when TARGET is an image.
    --help=false,-h           Don't display this help message.
    --version=false           Don't display version info.
```

Supplements [```docker cp```](https://docs.docker.com/engine/reference/commandline/cp/) by:
  * Facilitating image creation or adaptation by simply copying files.  When copying to an existing image, its state is unaffected, as copy preserves its immutability by creating a new layer.
  * Enabling the specification of mutiple copy sources, including other images, to improve operational alignment with linux [```cp -a```](https://en.wikipedia.org/wiki/Cp_%28Unix%29) and minimize layer creation when TARGET refers to an image.
  * Supporting the direct expression of copy semantics where SOURCE and TARGET arguments concurrently refer to containers.
 
#### Copy Semantics
```dkrcp``` relies on ```docker cp```, therefore, ```docker cp```'s [documentation](https://docs.docker.com/engine/reference/commandline/cp/) describes much of the expected behavior of ```dkrcp```, especially when specifying a single SOURCE argument.  Due to this reliance ```docker cp``` explinations concerning:
  * tarball streams ' - ',
  * container's root directory treated as the current one when copying to/from relative paths,
  * working directory of ```docker cp``` anchoring relative host file system paths,
  * desire to mimic ```cp -a``` recursive navigation and preserve file permissions,
  * ownership UID/GID permission settings when copying to/from containers, 
  * use of ':' as means of delimiting a container UUID/name from its associated path,
  * inability to copy certain files,

are all applicable to ```dkrcp```.

However, the following tabular form offers an equivalent description of copy behavior but visually different than the documention associated to ```docker cp```.

|         | SOURCE File  | SOURCE Directory | [SOURCE Directory Content](https://github.com/WhisperingChaos/dkrcp/blob/master/README.md#source-directory-content-an-existing-directory-path-appended-with-) | SOURCE Stream |
| :--:    | :----------: | :---------:| :----------: | :----------: |
| **TARGET exists as file.** | Copy SOURCE content to TARGET. | Error |Error | Error |
| **TARGET name does not exist but its parent directory does.** | TARGET File name assumed. Copy SOURCE content to TARGET.| TARGET directory name assumed. Create TARGET directory. Copy SOURCE content to TARGET. | Identical behavior to adjacent left hand cell. | Error |
| **TARGET name does not exist, nor does its parent directory.** | Error | Error | Error | Error|
| **TARGET exists as directory.** | Copy SOURCE to TARGET. | Copy SOURCE to TARGET. | Copy SOURCE content to TARGET. | Copy File/Directory to TARGET. |
| **[TARGET assumed directory](https://github.com/WhisperingChaos/dkrcp/blob/master/README.md#target-assumed-directory-the-rightmost-last-name-of-a-specified-path-suffixed-by---it-is-assumed-to-reference-an-existing-directory) doesn't exist but its parent directory does.** | Error | Copy SOURCE content to TARGET. | Copy SOURCE content to TARGET. | Error |

######TARGET assumed directory: The rightmost, last, name of a specified [path](https://en.wikipedia.org/wiki/Path_%28computing%29) suffixed by '/' - indicative of a directory.

######SOURCE Directory Content: An existing directory path appended with '/.'.

The multi-SOURCE copy semantics simply converge to the row labeled: '**TARGET exists as directory.**' above.  In this situation any SOURCE type, whether it a file, directory, or stream is successfully copied, as long as the TARGET refers to a preexisting directory, otherwise, the operation fails.

##### Images as SOURCE/TARGET
A double colon '```::```' delimiter classifies the file path as referring to an image, differenciating it from the single one denoting a container reference.  Therefore, an argument referencing an image involving a tag would appear similar to: '```repository_name:tag::etc/hostname```'.

When processing image arguments, ```dkrcp``` perfers binding to images known locally to Docker Engine that match the provided name and will ignore remote ones unless directed to search for them.  To include remote registries, specify the option: ```--ucpchk-reg=true```.  Enabling this feature will cause ```dkrcp``` to initiate ```docker pull``` iff the specified image name is not locally known.  Note, when enabled, ```ucpchk-reg``` applies to both SOURCE and TARGET image references. Therefore, in situations where the TARGET image name doesn't match a locally existing image but refers to an existing remote image, this remote image will be pulled and become the one referenced by TARGET.

Since copying to an existing TARGET image first applies this operation to a derived container (an image replica), its effects are "reversible".  Failures involving existing images simply delete the derived container leaving the repository unchanged.  However, when adding a new image to the local repository, the repository's state is first updated to reflect a [```scratch```](https://docs.docker.com/engine/userguide/eng-image/baseimages/#creating-a-simple-base-image-using-scratch) version of the image.  This ```scratch``` image is then updated in the same way as any existing TARGET image but if a failure occurs during the copy process, both the container and ```scratch``` image are removed, reverting the local repository's state.

######Copy *from* an *existing image*:
  * Convert the referenced image to a container via [```docker create```](https://docs.docker.com/engine/reference/commandline/create).
  * Copy from this container using ```docker cp```.
  * Destroy this container using ```docker rm```.

######Copy *to* an *existing image*:
  * Convert the referenced image to a container via ```docker create```.
  * Copy to this container using ```docker cp```.
    * When both the SOURCE and TARGET involve containers, the initial copy strategy streams ```docker cp``` output of the SOURCE container to a receiving ```docker cp``` that accepts this output as its input to the TARGETed container.  As long as the TARGET references an existing directory, the copy succeeds completing the operation.  However, if this copy strategy should fail, a second strategy executes a series of ```docker cp``` operations.  The first copy in this series, replicates the SOURCE artifacts to a temporary directory in the host environment executing ```dkrcp```.  A second ```docker cp``` then relays this SOURCE replica to the TARGET container.
  * If copy succeeds, ```dkrcp``` converts this container's state to an image using [```docker commit```](https://docs.docker.com/engine/reference/commandline/commit/).  
    * Specifying an image name as a TARGET argument propagates this name to ```docker commit``` superseding the originally named image.
    * When processing multiple SOURCE arguments, ```dkrcp``` delays the commit until after iterating over all of them.
  * If copy fails, ```dkrcp``` bypasses the commit.
  * Destroy this container using ```docker rm```.

######Copy *to create* an *image*:
  * Execute a ```docker build``` using [```FROM scratch```](https://docs.docker.com/engine/userguide/eng-image/baseimages/#creating-a-simple-base-image-using-scratch).
  * Continue with [Copy *to* an *existing image*](https://github.com/WhisperingChaos/dkrcp/blob/master/README.md#copy-to-an-existing-image).

####Install
#####Dependencies
  * GNU Bash 4.0+
  * Docker Engine 1.8+

#####Development Environment

  * Ubuntu 12.04
  * GNU Bash 4.2.25(1)-release
  * Docker Engine 1.9.1

#####Instructions

  * Select/create the desired directory to contain this project's git repository.
  * Use ```cd``` command to make this directory current.
  * Depending on what you wish to install execute:
    * ```git clone``` to copy entire project contents including the git repository.  Current master which may include untested features.
    * [```git archive```](https://www.kernel.org/pub/software/scm/git/docs/git-archive.html) to copy only the necessary project files without the git repository.  Archive can be selective by specifying tag or branch name.
    *  wget https://github.com/whisperingchaos/dkrcp/zipball/master creates a zip that includes only the project files without the git repository.  Current master branch which may include untested features.
  * Selectively add the 'dkrcp' alias by running the provided     


#### Why?
  * Promotes smaller images and potentially minimizes their attack surface by selectively copying only those resources required to run the containerized application.
    * Although special effort has been applied to minimize the size of Official Docker Hub images, the inability of Docker's [builder](https://github.com/docker/docker/tree/master/builder) component to separate build time produced artifacts and their required dependencies continues to pollute the runtime image with unnecessary artifacts increasing the runtime container's attack surface.  For example, images requiring build tool chains, like golang and C++, incorporate compilers, linkers, dependent libraries, ... into their files systems.  At best these tool chain resources can be 'logically' removed from the file system through their deletion.  These and other file system artifacts employed by the build, like C++ object files, will remain accessible in the runtime file system if not properly removed by the build process.  
  * Facilitates manufacturing images by piplines that gradually evolve either toward or away from their reliance on Dockerfiles.
    *  To accelerate the adoption of Docker containers, strategies dkrcp can enable a strategy increace developers understanding of Docker through the measured adoption by  encapsulating build tool chains reqired by their application into Docker containers.  
  * Encapsulates the reliance on and encoding of several Docker CLI calls to implement the desired functionality insulating automation incorporating this utility from potentially future improved support by Docker community members through dkrcp's interface.

###License

The MIT License (MIT) Copyright (c) 2015-2016 Richard Moyse License@Moyse.US

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions: The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

###Legal Notice

Docker and the Docker logo are trademarks or registered trademarks of Docker, Inc. in the United States and/or other countries. Docker, Inc. and other parties may also have trademark rights in other terms used herein.
