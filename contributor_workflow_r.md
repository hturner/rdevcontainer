# Contributor workflow for modifying R code on Ubuntu

This steps through the workflow of making and testing a change to an R function in base R on Ubuntu.

This is the traditional workflow, but using RStudio for debugging.

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
   Use debugging tools in R to work out the root cause (not worked through here!). In this case, the bug is in `stats::contrasts()`. 
 5. Edit the source code of `stats::contrasts()` to implement the proposed fix. The source code can be found in  `~/Downloads/R/src/library/stats/R/contrasts.R`. Before fix:
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

TODO: outline alternative GitHub workflow, which will allow us to run tests on multiple platforms.
