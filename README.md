# Description

This is the OctoPress-based site for <http://alister.github.oom>.

# Use of Git:

New posts should be written into the 'source' branch. This can be pushed on demand to the repository.

There is a remote branch: octopress. This can fetch the latest Octopress source, and from there can be 
merged into the 'source' branch.

# To upload the site to http://alister.github.oom

    bundle exec rake deploy # generate and upload to Github page
