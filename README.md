# Roughbook
Why we created DSL when jenkins file exist
There are some options in ahop-job in inova instance like recursive or snapshot dependencies. There are not pre defined api in Jenkins file
So DSL is another approach with which we can create a script for our Jenkins job.
# Issues observed:
Once we increased our Ram size from 512MB to 1GB, sample job was able to progress but failed at nabu step that is failing to find a file that should be part of Repository
Cloudfees nabu folder is not getting cloned so we manually cloned the reposiotry to get the build succeed.
