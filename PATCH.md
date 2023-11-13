# Patched Hash App code

## IMPORTANT before using this patched code, please read the instructions [below](#instructions-to-patch-code).

## Changes

- [Application code](./src/asd-project2-base/src/main/java/protocols/app/HashApp.java) changed to include support for two architectures, one using SMR + Paxos-based agreement, and one based on ABD Quorum for performing read/write operations directly.
- Modified [Main.java](./src/asd-project2-base/src/main/java/Main.java) to support multiple application architectures
- Very [incomplete ABD stubs](./src/asd-project2-base/src/main/java/protocols/app/HashApp.java#L15-L20) for integrating existing ABD code.
- Modified [config file](./src/asd-project2-base/config.properties) to specify replication strategy

## Instructions to patch code

**THIS IS A HIGHLY SENSITIVE OPERATION THAT CAN RESULT IN PROBLEMS IF YOU GET IT WRONG. MAKE SURE YOU BACK UP THE WORK THAT YOU HAVE ALREADY DONE.**

Following the instructions below should allow you to apply the patch in a safe way. If at any point you go wrong, you can always start again.

1. Push your latest changes to your online repository.

```bash
$ git push origin main
```

2. **Leave** the local directory containing your current repository.

```bash
$ cd ../
```

3. Make a new clone of your project repository.

```bash
$ git clone <existing_project_url> patch-repo
$ cd patch-repo
```

4. Set a new remote corresponding to this repository.

```bash
$ git remote add upstream git@github.com:MEI-ASD-2023/project-phase-02-patch.git
$ git fetch upstream
```

5. Attempt to merge in patches.

```bash
$ git merge upstream/main # you can also use `git rebase upstream/main`
```

6. At this point, if you have modified some of the code in the modified files, you will have conflicts to deal with, this is **normal**. You can fix these in the appropriate way, by applying your changes on top of the patched code. Once you have fixed each file with conflicts, if you used `git merge` you should commit the merge changes.

```bash
git commit -m "merge commit"
```

If you used `git rebase` you should continue:

```bash
git rebase --continue
```

7. You can then push the patched code to your original repository. If you used `git rebase` above, you may need to use the `-f` to *force push*, but use this ***carefully*** as it rewrites history.

```bash
git push origin master
```


