<!-- Karel Notes -->

## SETUP NOTES

1. You probably need to download Rstudio Desktop for mac. That's the most likely the best coding environment for R.

You also need to download some extra libraries, and there are a lot of dependencies for these. You will have to run this command in your terminal.

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

This will install brew, which you can use to install Gdal and udunits
```sh
brew install udunits
brew install gdal
```

Run these commands in your terminal.

** Now you can install the necessary R packages. This can take a loong time
install.packages('tidyverse')

** NOTE, they don't tell you 'usmap' has dependencies that you need to install.
install.packages(c('sf', 'units', 'usmapdata'))
install.packages('usmap')
install.packages('knitr')

2.  *knit* means use the package `knitr` if I'm not mistaken. You may have to install some extra packages to get it working.


### OVERALL LAB NOTES

The lab is kinda simple up until question 4, try to answer the questions yourself without any help from AI, but if you get stuck you can use AI. Question 4 gets more involved with merging the dataset, which becomes more data-science-ey and less programming focused. But that's not a big issue. 

My thoughts on what's important to remember from this lab.

1. How to access a variables in dataframe (using the `$` symbol)
2. What commands get you the `mean`, `standard deviation`, `min`, `max` etc.. How do you call those functions?
    - This is important because sometimes you think it's UVL$PctMoveIn.sd() when it's actually sd(UVL$PctMoveIn). In computer science, you could argue the standard deviation is a property of the list of values, so should be a function of the list, but (also programmatically) it's cheaper to just have one universal function for standard deviation. This is just a little fun fact.
3. You should also memorize the functions for different plots. (i.e barplot, plot, abline, etc.) `plot_usmap` is a confusion function, you don't need to memorize that one.


For Q5, when they ask you to plot the data on a US map, the plot doesn't tell you anything new that you don't already know from the scatter plot. Don't necessarily know why they make you do that.



