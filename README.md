# fbs-release-github-actions

This is a brief guide on 
how to automatically build a python application using 
[fbs](https://build-system.fman.io/)
and how to automatically create installer on all platforms
and a GitHub release using 
[GitHub Actions](https://github.com/features/actions).

**`fbs pro`**: If you have purchased `fbs pro`,
check out [this documentation](fbs_pro_instructions.md)
on how to modify the following workflows
to use the download link that you have received
in a secure manner.

## Project structure

### Python program

The program that is bundled with fbs here and released
is the full version from the 
[fbs tutorial](https://github.com/mherrmann/fbs-tutorial).
This program lives in the `src` folder.

### Python requirements

Requirements, including fbs and PyQt are stored in the
`requirements.txt` file. It is important that the file
given here contains all requirements needed
to successfully compile the project.
More detailed information,
including how to freeze a requirement file
from your current virtual environment
can be found 
[in this blog post](https://medium.com/@boscacci/why-and-how-to-make-a-requirements-txt-f329c685181e).

### Text for the next release

The text for the next release should be stored
in a file named `release_text.md`,
which lives in the root folder of the GitHub repository.
Please see below in the section
where the workflow files are discussed in detail
for more information.

### bump2version

This excellent project is used to version bump the package.
More information can be found
[here](https://github.com/c4urself/bump2version).

For a new release,
the version bump will increment the files
that are mentioned in the `.bumpversion.cfg`
configuration file.
Here, only one file gets bumped, namely:

    src/build/settings/base.json
    
This file is the fbs base file
that will compile your program with 
the correct version.

As you can see in the configuration file,
bump2version will also create a new tag
as well as a commit.
This is important since our release creating workflow
should only run when a new tag is pushed to the repository.
This tag will then be used to create the release.

### Workflows

All `.yml` workflow files are stored for easy access
in the `workflows` folder.
In order for GitHub to run your action
after a tag is pushed,
the workflow file that you want to use must be located in:

    .github/workflows/
    
## The workflow files

Two kinds of workflow files were created:

 - Releases for one single operating system only
 - Releases for all operating systems
 
The difference between these workflow files is
that single OS releases use the `actions/upload-release-asset`
to add the assets to the release.
Multi-OS releases use the `svenstaro/upload-release-action`
for this asset addition.
You can read up
[here](https://github.com/actions/upload-release-asset)
and [here](https://github.com/svenstaro/upload-release-action)
about the differences.
While the latter action is simpler to implement for the mulit-OS case,
the former is an action that is directly supplied by GitHub Actions.
 
*Note*: Only Ubuntu, macOS Catalina, 
macOS BigSur, 
and Windows Server 2019 releases
are currently possible,
since these are the potential environments that GitHub Actions run under.
[Check here](https://docs.github.com/en/actions/reference/virtual-environments-for-github-hosted-runners#supported-runners-and-hardware-resources)
to see what runners are available.

Let's go in detail through the Linux single OS release first.
The other workflows will be discussed where necessary,
i.e., to point out differences.

### Ubuntu single-OS release

The workflow file that goes with this description
can be found in:

    workflows/release-ubuntu.yml

Every action in this tutorial starts with:

```yaml
on:
  push:
    # Sequence of patterns matched against refs/tags
    tags:
    - 'v*' # Push events to matching v*, i.e. v1.0, v20.15.10
```

This means that the action is executed whenever a tag is pushed.
The tag must follow the general GitHub scheme, 
e.g., `v0.0.5`.
In the action, the part `'v*'` matches these tags.

```yaml
name: Release Ubuntu
```

This next line is the name of the GitHub Action.
This will also be the name that you get displayed
when you click on the workflows tab in your repository.

The next line named `jobs` now starts the individual jobs
that should be performed whenever the action is triggered.
For a single-OS release creation, this is only one job.

```yaml
build:
    name: Upload Release Asset
    runs-on: ubuntu-16.04
    strategy:
      matrix:
        python-version: [3.6]
```
         
The first (and here only) job is named build.
First we supply the name of the job,
then the operating system the job runs on.
As pointed out in the 
[PyInstaller docs](https://pyinstaller.readthedocs.io/en/stable/usage.html#making-gnu-linux-apps-forward-compatible),
we want to use the oldest Linux available
for GitHub Actions.
Then we define the python versions that we want to use.
For fbs, as currently required,
we will use the latest supported version,
i.e., 3.6.
Now the job is broken up into individual steps,
denoted by the title `steps` in line 16.

  - name: Checkout code
    uses: actions/checkout@v2
        
This first step simply uses a GitHub supplied action
to check out our repository.

```yaml
- name: Install dependencies
run: |
  sudo apt-get install ruby ruby-dev rubygems build-essential
  sudo gem install --no-document fpm
  fpm --version
```

The second step will install the Linux dependencies.
As you can see in the 
[fbs tutorial](https://github.com/mherrmann/fbs-tutorial),
[fpm](https://github.com/jordansissel/fpm)
is installed following
[these instructions](https://fpm.readthedocs.io/en/latest/installing.html).

```yaml
- name: Set up Python ${{ matrix.python-version }}
  uses: actions/setup-python@v2
  with:
    python-version: ${{ matrix.python-version }}
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
```
      
The third and fourth step now first set up the python environment,
again using a GitHub provided action,
and then install the dependencies
that we have stored in the `requirements.txt` file.

```yaml
- name: Run fbs and freeze application
  run: |
    fbs freeze
    fbs installer
```

Step five now runs `fbs freeze` to create the package and then
`fbs installer` to create the installer. 

```yaml
- name: Create Release
  id: create_release
  uses: actions/create-release@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    tag_name: ${{ github.ref }}
    release_name: Release ${{ github.ref }}
    # body: ""  # release message, alternative to body_path
    body_path: release_text.md  # uncomment if not used
    draft: false
    prerelease: false
```
       
The sixth step creates the release.
More information and options about this can be found
[here](https://github.com/actions/create-release).
Important here to note is the `body_path` variable.
If a markdown file should be used for the release,
as used here,
pass this variable the filename (here `release_text.md`).
Alternatively you can directly supply the text for the release
using the (here commented out) `body` variable.

```yaml
- name: Upload Release Asset
  id: upload-release-asset
  uses: actions/upload-release-asset@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    upload_url: ${{ steps.create_release.outputs.upload_url }} # This pulls from the CREATE RELEASE step above, referencing it's ID to get its outputs object, which include a `upload_url`. See this blog post for more info: https://jasonet.co/posts/new-features-of-github-actions/#passing-data-to-future-steps
    asset_path: target/fbs_tutorial_app.deb
    asset_name: fbs_tutorial_app.deb
    asset_content_type: application/deb
```

This final step now adds the created binaries to the release.
Note that you must specify the correct path and file names here
that fbs will create! 
`asset_path` tells the action where the binary to be used can be found,
`asset_name` tells it the name it should use in the release.

### macOS single-OS release

The workflow file that goes with this description
can be found in:

    workflows/release-macos.yml

The macOS release works almost identical.
No dependencies aside from python are required.
**Note** that the installer generated
has a different file ending.

### Windows single-OS release

The workflow file that goes with this description
can be found in:

    workflows/release-windows.yml

Again, the Windows release looks almost identical.
[NSIS](https://nsis.sourceforge.io/Main_Page)
is already installed in GitHub windows runners.
**Note** that the binaries for the installers
have an added `Setup`
and use the file ending `.exe`.

### Multi-OS releases

The workflow file that goes with this description
can be found in:

    workflows/release-multi-os.yml

The general structure of each release is similar
to the ones described above.
However, multiple `build` jobs are included,
one for every operating system of interest.

Creating the release itself is done much earlier now
in order to have the relase created by the time
the first job adds assets.
A release can furthermore only be
created once.
Here it is created in the `build-linux` job.

To upload the binaries to the asset,
the following action is used:

```yaml
- name: Upload binaries to release
  uses: svenstaro/upload-release-action@v2
  with:
    repo_token: ${{ secrets.GITHUB_TOKEN }}
    file: target/fbs_tutorial_app.deb
    asset_name: fbs_tutorial_app.deb
    tag: ${{ github.ref }}
    overwrite: true
```

Here, `file` is the name for the binary to be added
and `asset_name` is the name of the file that will be
in the release.
More information on this release can be found
[here](https://github.com/svenstaro/upload-release-action).

## Releases

The [tutorial releases](https://github.com/trappitsch/fbs-release-github-actions/releases)
were all created with the above mentioned workflows. 
Release v0.0.2, v0.0.3, and v0.0.4 contain the
Ubuntu, Windows, and macOS versions.
Release v0.0.5 contains the multi-OS release.

## Questions, comments, ...

If you have questions, comments, etc.
on this tutorial,
please raise an issue. 
Any ideas for features / enhancements?
I'm sure you do! 
Please file a feature request as well.

## License

MIT, see [license file](LICENSE).

## Acknowledgements

- [@Kastakin](https://github.com/Kastakin) for pointing out 
  that the oldest available Linux version should be used
  (Issue [#2](https://github.com/trappitsch/fbs-release-github-actions/issues/2))