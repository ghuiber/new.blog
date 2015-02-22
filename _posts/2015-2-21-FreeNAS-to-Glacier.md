---
layout: post
title: Backing up FreeNAS to AWS Glacier
tags:
- FreeNAS
- AWS
---
On my [FreeNAS home server](http://enoriver.net/index.php/2014/01/11/freenas-works-as-advertised/) I added a new [jail](http://doc.freenas.org/9.3/freenas_jails.html), called it `awsboss`. I also [added a new ZFS dataset](http://doc.freenas.org/9.3/freenas_storage.html#create-dataset), called it `storage`. I added this data set to `awsboss` as `/mnt/storage` and I also made it into an [AFP share](http://doc.freenas.org/9.3/freenas_sharing.html?highlight=share#apple-afp-shares) so it could be mounted by my Mac clients.

The `storage` database will receive from Mac clients in my house various tarballs of files that I want to keep in deep storage on Glacier, such as old pictures. Then `awsboss` will upload them to Glacier from there.

The test file for this exercise is `pics2011.tgz`, a 2G tarball of everything that's in the `~/Pictures/iPhoto Library/Masters/2011` folder on my Mac. If this works, then at the end of every year I'll just make another tarball with all of the pictures and videos taken that year and send it along. These tarballs might as well go into a vault of their own, named something like `iPhotoGabi`, which I created at the [AWS Console](https://console.aws.amazon.com).

### Items 1-4 below happen at the root prompt on `awsboss`:

1. I installed and configured the [AWS Command Line Interface (CLI)](http://aws.amazon.com/cli/) using the instructions for Mac/Linux. For this, I had to install pip first using the [`get-pip.py` script](https://pip.pypa.io/en/latest/installing.html). Configuration instructions for AWS CLI are [here](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) and they include a link to getting started with Identity and Access Management (IAM), which is where you set up the access key pairs that the CLI needs. If you follow the configuration instructions you're in effect writing two files: `~/.aws/credentials` and `~/.aws/config`.

2. I installed [boto](http://docs.pythonboto.org/en/latest/getting_started.html), which also needs AWS credentials in the `~/.boto` file, but these are slightly different from those stored in `~/.aws/credentials`. The latter can include key pairs for multiple users, each pair under a header with the `[userName]` in brackets as shown. The `~/.boto` file is user-specific, so it includes only one pair under the header `[Credentials]`. So making `~/.boto` a symlink to `~/.aws/credentials` won't work. I tried.

3. As of this writing `boto` comes with a `glacier` command of its own, so now things should be easy. I expect that typing `glacier upload iPhotoGabi /mnt/storage/photos2011.tgz` will do the job.

4. I was wrong. There was a bit of a problem. In response to the `glacier upload` command above I got a bunch of Python error references on screen, the last of which was `boto.glacier.exceptions.UploadArchiveError: An error occurred while uploading an archive: 'ascii' codec can't decode byte 0x8b in position 1: ordinal not in range(128)`. This may look obscure, but it's something you can Google. I ended up on Stack Overflow and fixed two things: I created the file `/usr/local/lib/python2.7/site-packages/sitecustomize.py` to change my default encoding to Unicode as shown [here](http://stackoverflow.com/questions/21129020/how-to-fix-unicodedecodeerror-ascii-codec-cant-decode-byte); and then I fixed line 1276 of `/usr/local/lib/python2.7/site-packages/boto/glacier/layer1.py` as shown [here](https://github.com/tsileo/bakthat/issues/72).

There you have it. As of this writing, that 2G upload is in progress, off the `awsboss` jail, and my Mac client is free to do other things. God, I hope this works.

### The next day

The upload completed and the `iPhotoGabi` vault now shows two items and a total size of 2.04GiB. One of the two items may be a small pdf I uploaded just for testing. To get a listing of what exactly is in a vault is a two-step process: first you initiate an inventory retrieval job, then you wait. When the retrieval job completes you can collect the output. You know that the job completed when status in the response to 
`$ aws glacier --account-id='-' --vault-name='iPhotoGabi' list-jobs` changes from `InProgress` to `Succeeded`. 

That response contains one line per job, and in each line there is a very long alphanumeric string which is the unique job ID. That's the one to pass in the command 
```
$ aws glacier --account-id='-' --vault-name='iPhotoGabi' --job-id="longstringhere" get-job-output output.json
```

The file `output.json` in the working directory contains a JSON object that lists archive ID's and descriptions. 

The directions for this are on [Reddit](http://www.reddit.com/r/aws/comments/2ujfoh/any_tools_for_generating_a_list_filetxtcsv_or/).

Before this retrieval job completes there's not much I can do, but using the syntax in the examples on that same Reddit thread, I can at least describe my vault:
```
$ aws glacier --account-id='-' --vault-name='iPhotoGabi' describe-vault
```

All of this only worked after I re-ran `aws configure` and entered the key pairs and region for the default user. Either the AWS CLI won't recognize specific IAM profiles, or they must be passed along as an extra parameter. That's fine. The default user works for me.

### Conclusion

This seems to work. I uploaded a 2G tarball and a small pdf using boto's `glacier upload` facility within a reasonable amount of time from a FreeNAS jail, and managed to retrieve them both using the AWS CLI.
