# Git Resource

Tracks the commits in a [git](http://git-scm.com/) repository.


## Source Configuration

* `uri`: *Required.* The location of the repository.

* `branch`: *Required.* The branch to track.

* `private_key`: *Optional.* Private key to use when pulling/pushing.
    Example:
    ```
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    ```
    Note: You can also use pipeline templating to hide this private key in source control. (For more information: https://concourse.ci/fly-set-pipeline.html)

* `username`: *Optional.* Username for HTTP(S) auth when pulling/pushing.
  This is needed when only HTTP/HTTPS protocol for git is available (which does not support private key auth)
  and auth is required.

* `password`: *Optional.* Password for HTTP(S) auth when pulling/pushing. 

   Note: You can also use pipeline templating to hide this password in source control. (For more information: https://concourse.ci/fly-set-pipeline.html)

* `paths`: *Optional.* If specified (as a list of glob patterns), only changes
  to the specified files will yield new versions from `check`.

* `ignore_paths`: *Optional.* The inverse of `paths`; changes to the specified
  files are ignored.

  Note that if you want to push commits that change these files via a `put`,
  the commit will still be "detected", as [`check` and `put` both introduce
  versions](https://concourse.ci/pipeline-mechanics.html#collecting-versions).
  To avoid this you should define a second resource that you use for commits
  that change files that you don't want to feed back into your pipeline - think
  of one as read-only (with `ignore_paths`) and one as write-only (which
  shouldn't need it).

* `skip_ssl_verification`: *Optional.* Skips git ssl verification by exporting
  `GIT_SSL_NO_VERIFY=true`.

* `tag_filter`: *Optional.* If specified, the resource will only detect commits
  that have a tag matching the specified expression. Patterns are
  [glob(7)](http://man7.org/linux/man-pages/man7/glob.7.html) compatible (as
  in, bash compatible).

* `git_config`: *Optional.* If specified as (list of pairs `name` and `value`)
  it will configure git global options, setting each name with each value.

  This can be useful to set options like `credential.helper` or similar.

  See the [`git-config(1)` manual page](https://www.kernel.org/pub/software/scm/git/docs/git-config.html)
  for more information and documentation of existing git options.

* `disable_ci_skip`: *Optional.* Allows for commits that have been labeled with `[ci skip]` or `[skip ci]`
   previously to be discovered by the resource.

* `commit_verification_keys`: *Optional.* Array of GPG public keys that the
  resource will check against to verify the commit (details below).

* `commit_verification_key_ids`: *Optional.* Array of GPG public key ids that
  the resource will check against to verify the commit (details below). The
  corresponding keys will be fetched from the key server specified in
  `gpg_keyserver`. The ids can be short id, long id or fingerprint.

* `gpg_keyserver`: *Optional.* GPG keyserver to download the public keys from.
  Defaults to `hkp:///keys.gnupg.net/`.

* `all_branches`: *Optional*. If set to `true` then fetch all branches, not
  just a single specific branch. Particularly useful if you switch branches
  before pushing to your remote git repository.

### Example

Resource configuration for a private repo:

``` yaml
resources:
- name: source-code
  type: git
  source:
    uri: git@github.com:concourse/git-resource.git
    branch: master
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    git_config:
    - name: core.bigFileThreshold
      value: 10m
    disable_ci_skip: true
```

Fetching a repo with only 100 commits of history:

``` yaml
- get: source-code
  params: {depth: 100}
```

Pushing local commits to the repo:

``` yaml
- get: some-other-repo
- put: source-code
  params: {repository: some-other-repo}
```

## Behavior

### `check`: Check for new commits.

The repository is cloned (or pulled if already present), and any commits
from the given version on are returned. If no version is given, the ref
for `HEAD` is returned.

Any commits that contain the string `[ci skip]` will be ignored. This
allows you to commit to your repository without triggering a new version.

### `in`: Clone the repository, at the given ref.

Clones the repository to the destination, and locks it down to a given ref.
It will return the same given ref as version.

Submodules are initialized and updated recursively.

#### Parameters

* `depth`: *Optional.* If a positive integer is given, *shallow* clone the
  repository using the `--depth` option. Using this flag voids your warranty.
  Some things will stop working unless we have the entire history.

* `submodules`: *Optional.* If `none`, submodules will not be
  fetched. If specified as a list of paths, only the given paths will be
  fetched. If not specified, or if `all` is explicitly specified, all
  submodules are fetched.

* `disable_git_lfs`: *Optional.* If `true`, will not fetch Git LFS files.

#### GPG signature verification

If `commit_verification_keys` or `commit_verification_key_ids` is specified in
the source configuration, it will additionally verify that the resulting commit
has been GPG signed by one of the specified keys. It will error if this is not
the case.

### `out`: Push to a repository.

Push the checked-out reference to the source's URI and branch. All tags are
also pushed to the source. If a fast-forward for the branch is not possible
and the `rebase` parameter is not provided, the push will fail.

#### Parameters

* `repository`: *Required.* The path of the repository to push to the source.

* `rebase`: *Optional.* If pushing fails with non-fast-forward, continuously
  attempt rebasing and pushing.

* `tag`: *Optional.* If this is set then HEAD will be tagged. The value should be
  a path to a file containing the name of the tag.

* `only_tag`: *Optional.* When set to 'true' push only the tags of a repo.

* `tag_prefix`: *Optional.* If specified, the tag read from the file will be
prepended with this string. This is useful for adding `v` in front of
version numbers.

* `branch`: *Optional* If this is set then push to the given branch. The
  value should be a path to a file containing the name of the branch. Overrides
  the branch name given in the main resource definition. This should be paired
  with the `all_branches` source configuration (see above).
* `force`: *Optional.* When set to 'true' this will force the branch to be
pushed regardless of the upstream state.

* `annotate`: *Optional.* If specified the tag will be an
  [annotated](https://git-scm.com/book/en/v2/Git-Basics-Tagging#Annotated-Tags)
  tag rather than a
  [lightweight](https://git-scm.com/book/en/v2/Git-Basics-Tagging#Lightweight-Tags)
  tag. The value should be a path to a file containing the annotation message.
