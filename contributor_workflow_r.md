# Contributor workflow for modifying R code on Ubuntu

This steps through the workflow of making and testing a change to an R function in base R on Ubuntu.

This is the traditional workflow, but using RStudio for debugging.

## From first build to creating a patch

We will use [Bug 17616](https://bugs.r-project.org/show_bug.cgi?id=17616) as an example, 
where the reported issue can be fixed with changing a single line of R code.

Thes instructions were tested on Ubuntu 22.04.02

1. Install the [prerequisites on Ubuntu](https://contributor.r-project.org/rdevguide/GetStart.html#ubuntu).
2. Follow the instructions for [building R on Linux](https://contributor.r-project.org/rdevguide/GetStart.html#ubuntu), with one exception
    - In Step 0, check out the source code for R 4.2.3, where Bug 17616 can be reproduced:  
        ```
        svn checkout https://svn.r-project.org/R/tags/R-4-2-3/ "$TOP_SRCDIR"
        ```
3. Open the built R in RStudio  
   ```
   export RSTUDIO_WHICH_R="$BUILDDIR/bin/R"
   cd "$TOP_SRCDIR"
   rstudio </dev/null &>/dev/null &
   ```
   Notes:
   * `cd "$TOP_SRCDIR"` should mean that rstudio opens with no project open and with `"TOP_SRCDIR"` as the working directory. This didn't work for me, though `rstudio /home/Downloads/bin/R </dev/null &>/dev/null &` does
   * The `</dev/null &>/dev/null &` detaches RStudio from the terminal so we can use both independently.
4. Run the [reprex](https://bugs.r-project.org/show_bug.cgi?id=17616#c2) to confirm the bug. Expected output with bug (the second parameter label has the number of the factor level, "2", rather than the label of the factor level, "chilled") :
   ```
   > lm(uptake~C(Treatment, contr.treatment), CO2)
   Call:
   lm(formula = uptake ~ C(Treatment, contr.treatment), data = CO2)

   Coefficients:
                      (Intercept)  C(Treatment, contr.treatment)2  
                            30.64                           -6.86
   ```
   Use debugging tools in R to work out the root cause (not worked through here!). In this case, the bug is in ``stats::`contrasts<-()` ``. 
 5. Edit the source code of ``stats::`contrasts<-()` `` to implement the proposed fix. The source code can be found in  `$BUILDDIR/src/library/stats/R/contrasts.R`. Before fix:
    ```
    if(is.function(value)) value <- value(nlevels(x))
    ```
    With fix:
    ```
    if(is.function(value)) value <- value(levels(x))
    ```
 6. Re-build the stats package (we only need to re-build the part we have modified).
    ```
    cd "$BUILDDIR/src/library/stats"
    make
    ```
 7. Restart R session in RStudio.

 8. Check fix has worked as expected by re-running the reprex. Expected output without bug 
    ```
    > lm(uptake~C(Treatment, contr.treatment), CO2)
    Call:
    lm(formula = uptake ~ C(Treatment, contr.treatment), data = CO2)

    Coefficients:
                             (Intercept)  C(Treatment, contr.treatment)chilled  
                                   30.64                                 -6.86 
    ```
 9. Create a patch showing what we have changed
    ```
    cd "$TOP_SRCDIR"
    svn update
    svn diff > patch.diff
    ```
 10. Run `make check` to check we have not broken any thing
     ```
     cd "$BUILDDIR"
     make check
     ```
   
After this, the patch can be submitted via R's Bugzilla for review.

## Updating R sources to start work on another bug

1. If necessary, move to the branch you want to work:
   ```
   export TOP_SRCDIR="$HOME/Downloads/R" # only needed if you have restarted the terminal
   cd $TOP_SRCDIR
   svn switch https://svn.r-project.org/R/trunk/
   ```
   Otherwise, update to the latest revision
   ```
   svn update
   ```
 2. Sync the recommended packages
    ```
    "$TOP_SRCDIR/tools/rsync-recommended"
    ```
 3. Configure and make R
    ```
    export BUILDDIR="$HOME/bin/R"
    cd "$BUILDDIR"
    "$TOP_SRCDIR/configure" --enable-R-shlib
    make
    ```
  4. Continue as from step 3 above
  
  Rebuilding R takes a lot less time than starting from scratch ~5-10 mins vs ~30 mins.
    
## GitHub equivalent (for latest R-devel only)

Here we consider the latest R-devel only, assuming that we are making a real change that we will want to test using 
continuous integration on multiple platforms.

To illustrate an old bug we could go back to an old commit and modify the code, but we wouldn't work want to create a PR from this 
to the latest commit and hence can't demo testing a fix and creating a patch this way. 

1. Install the [prerequisites on Ubuntu](https://contributor.r-project.org/rdevguide/GetStart.html#ubuntu).
2. Fork GitHub mirror of the R svn repo: https://github.com/r-devel/r-svn/tree/master
3. Clone the fork locally (e.g. by creating an RStudio project)
4. In the terminal, set `TOP_SRCDIR` to where you have cloned the repo and use `sed` to modify the make file so that 
   instead of using `svn info` to get the SVN version number (which would only work for an svn checkout), 
   the make file uses a script that infers the SVN version number from the information in the git repo.
   ```
   export TOP_SRCDIR="$HOME/Repos/r-svn"
   sed -i.bak 's|$(GIT) svn info|./.github/scripts/svn-info.sh|' Makefile.in
   ```
   (a backup is stored in `Makefile.in.bak`).
5. Continue to build R as in steps 2 and 3 in last section.
6. Use the built R, modify source files and re-build componenents as in steps 3 to 8 in the first section.
7. Commit the changes you wish to make to the source files (not the change to `Makefile.in` or untracked files such as the tarballs of recommended packages) and push the commits to your fork on GitHub.
8. Open a PR from your fork back to https://github.com/r-devel/r-svn to trigger the continuous integration tests.
9. If the tests pass, create a patch by adding `.diff` to the URL of the PR.
