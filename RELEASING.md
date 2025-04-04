# Releasing

## Requirements
* [github client](https://github.com/cli/cli#installation)
* [gitchub_changelog_generator](https://github.com/github-changelog-generator)
* [github access token](https://github.com/github-changelog-generator/github-changelog-generator#github-token)

## Export changelog token

```bash
export CHANGELOG_GITHUB_TOKEN=token
```

## Set release information

```bash
export NEW_RELEASE=x.x.x
```

## Normal releases from master

### Create a branch for the release

```bash
git checkout master
git fetch upstream master
git rebase upstream/master
git checkout -b release/$NEW_RELEASE
```

### Update the version number

```bash
sed -i "0,/version = /{s/version = .*/version = \"${NEW_RELEASE}\"/}" pyproject.toml
```

### Update dependencies

```bash
uv sync --all-extras
uv lock --upgrade
uv sync --all-extras
```

### Run pre-commit and tests

```bash
uv run pre-commit run --all-files
uv run pytest
```

### Create release summary (skip for dev releases)

Write a short and understandable summary for the release.  Can include images.

#### Create $NEW_RELEASE milestone in github

If not already created

#### Create new issue linked to the milestone

```bash
gh issue create --label "release-summary" --milestone $NEW_RELEASE --title "$NEW_RELEASE Release Summary" --body "**Release highlights:**"
```

You can exclude the --body option to get an interactive editor or go into the issue on github and edit there.

#### Close the issue

Either via github or:

```bash
gh issue close ISSUE_NUMBER
```

### Generate changelog

Configuration settings are in `.github_changelog_generator`

#### For pre-release

EXCLUDE_TAGS will exclude all dev tags except for the current release dev tags.

Regex should be something like this `^((?!0\.9\.0)(.*dev\d))+`. The first match group negative matches on the current release and the second matches on releases ending with dev.

```bash
EXCLUDE_TAGS=${NEW_RELEASE%.dev*}; EXCLUDE_TAGS=${EXCLUDE_TAGS//"."/"\."}; EXCLUDE_TAGS="^((?!"$EXCLUDE_TAGS")(.*dev\d))+"
echo "$EXCLUDE_TAGS"
github_changelog_generator --future-release $NEW_RELEASE --exclude-tags-regex "$EXCLUDE_TAGS"
```

#### For production

```bash
github_changelog_generator --future-release $NEW_RELEASE --exclude-tags-regex 'dev\d$'
```

You can ignore warnings about missing PR commits like below as these relate to PRs to branches other than master:
```
Warning: PR 29 merge commit was not found in the release branch or tagged git history and no rebased SHA comment was found
```


### Export new release notes to variable

```bash
export RELEASE_NOTES=$(grep -Poz '(?<=\# Changelog\n\n)(.|\n)+?(?=\#\#)' CHANGELOG.md | tr '\0' '\n' )
echo "$RELEASE_NOTES"  # Check the output and copy paste if neccessary
```

### Commit and push the changed files

```bash
git commit --all --verbose -m "Prepare $NEW_RELEASE"
git push upstream release/$NEW_RELEASE -u
```

### Create a PR for the release, merge it, and re-fetch the master

#### Create the PR
```
gh pr create --title "Prepare $NEW_RELEASE" --body "$RELEASE_NOTES" --label release-prep --base master
```

#### Merge the PR once the CI passes

Create a squash commit and add the markdown from the PR description to the commit description.

```bash
gh pr merge --squash --body "$RELEASE_NOTES"
```

### Rebase local master

```bash
git checkout master
git fetch upstream master
git rebase upstream/master
```

### Create a release tag

Note, add changelog release notes as the tag commit message so `gh release create --notes-from-tag` can be used to create a release draft.

```bash
git tag --sign $NEW_RELEASE -m "$RELEASE_NOTES"  # to create an unsigned tag replace --sign with --annotate
git push upstream $NEW_RELEASE
```


### Approve the release workflow

This will automatically deploy to pypi
