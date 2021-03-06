---
permalink: /:categories/:title/
layout: post
title: "Multiple Names for Datasets in R Packages"
date: 2021-01-30 15:00:00.000000000 +00:00
type: post
published: true
status: publish
categories:
- projects
image: https://owenjonesuob.github.io/assets/featured/spiderman.jpg
tags:
- r
- package
- data
excerpt: "Can we use multiple names to refer to the same dataset, without including multiple copies of it in the package?"
---

> "I am known by many names, but you may call me... Tim." - **Graham Chapman**

---

Last year[^1] the [{mangoTraining}](https://github.com/MangoTheCat/mangoTraining) package got a fair bit of much-needed TLC, and we discovered a problem which at first glance didn't seem too tricky to solve, but which ended up leading down a very interesting rabbit hole.

[^1]: Yes, it's taken me nearly a year to write up this post. Please forgive me, I've been very busy surviving.


The package is primarily a "data package", i.e. a package which exists purely to provide datasets to users, and contains nearly 30 datasets of various sizes which are used in Mango's training courses and referred to in books such as "R in 24 Hours".

We tend to teach the latest best practices, so our training courses are updated on a regular basis.[^2] Unfortunately once a physical book has been released into the wild, it's not so easy to update it in the same way; so whenever we make changes to our material, we have to be careful not to break backwards compatibility!

[^2]: Keeping content up to date can be quite a lot of work in itself! So in fact it was only quite recently that we committed 100% to following [the tidyverse style guide](https://style.tidyverse.org/) wherever possible.


This is how we stumbled upon the "multiple names for datasets in R packages" problem. Suppose we have a dataset in {mangoTraining} which was historically called `demoData`, but we now favour snake_case rather than lowerCamelCase in our recent material: how can we _also_ use the name `demo_data` to refer to the dataset, so that we can use the new name in our new material without breaking compatibility with existing material?


And since we generally want to keep packages as small as possible[^3]: can we do this without including multiple copies of the same dataset in the package?

[^3]: While keeping packages small is a good idea anyway (especially if you're using version control!), an extra motivating factor here is the 5MB upload limit for CRAN packages as detailed on the [CRAN policies page](https://cran.r-project.org/web/packages/policies.html). You can imagine a situation where we want to use 2 different names for a 3MB dataset!


This allows us to set out three requirements which cover what we're trying to achieve:

1. The package must contain only a single copy of the full dataset
2. The dataset must be accessible using at least two different names
3. Each name must work correctly with `data()`, i.e. `data("x", package = "pkg")` must load an object called `x` into the session



## Adding more data files


I'm going to start by creating a new package, and then and immediately adding a dataset which I would like to include - which is incredibly simple these days, thanks to the marvellous [{usethis} package](https://github.com/r-lib/usethis)!

```r
 usethis::create_tidy_package("datapkg")
 
 # ... and then once the new session has opened:
 diamonds <- data("diamonds", package = "ggplot2")
 usethis::use_data(diamonds)
 ```

Following this, we can see that our dataset has been saved into the `data/` directory as a `.rda` file[^4], meaning our users can load this dataset into their session using `data("diamonds", package = "datapkg")`.

[^4]: As we would expect from {usethis}, this completely aligns with the rules and conventions on saving datasets within R packages, set out in [r-pkgs](https://r-pkgs.org/data.html) and elsewhere.

The name of the file has been created, quite sensibly, from the name of the R object we saved into it. Renaming the file isn't a good idea, because the object it contains will still have the old name! For example, if we manually renamed `data/diamonds.rda` to `data/diamantes.rda` and rebuilt/reinstalled our package, then if we ran

```r
data("diamantes", package = "datapkg")
```

we _would_ get our data loaded into our session; BUT it would still be in an object called `diamonds`! This is because `.rda` files store an entire R environment, possibly containing multiple objects: when we load that environment into our session, all the objects are just dumped into the session, exactly as they were when they were saved.[^5]

[^5]: It took me a REALLY long time to get my head around this at first - has anyone else had an experience like trying to run `diamantes <- data("diamonds")`, and then getting really frustrated when `diamantes` is not the data frame you were expecting? I found that so unintuitive. Actually nowadays when I need to save a single R object, I tend to use `.rds` rather than `.rda` for exactly that reason. However, `.rda` is the convention for R packages, so we'll stick with it... grrrr...


So if we want a different name for our dataset, we actually need to rename the object itself before saving it:

```r
diamantes <- diamonds
usethis::use_data(diamantes)
```

This is all useful context, but we haven't actually satisfied our first requirement yet! Our users _can_ now get either `diamonds` or `diamantes` from our package; but that's because the package contains the same dataset twice, in two separate `.rda` files (`diamonds.rda` and `diamantes.rda`). What we would ideally like to do is to have multiple names pointing towards the same underlying dataset, so that we only have to have one copy of that dataset in our package but it can be called in multiple ways.



## Multiple names for functions


It's really easy to do something along those lines with functions, because we can store a function in a variable and then just reassign that variable to our heart's content.[^6]

[^6]: This is because functions in R are "first-class objects", i.e. you can give them a name, save them in variables, pass them into other functions as parameters... I took this for granted until recently, when I found out that you _can't_ do this so easily in many languages - for example, Java!


To prove the point:

```r
f <- function(x, y) x+y
g <- f
g
```

{% highlight rout %}
function(x, y) x+y
{% endhighlight %}


This can be really useful when you want to implement a piece of functionality in just one place, but then to allow the user to access the same functionality via more than one name. Here's an example from {tibble}: the "type check" function for tibble objects is named `is_tibble()`, in line with tidyverse naming conventions; but the authors also provide an alias, `is.tibble()`, which matches base R naming conventions (`is.numeric()`, `is.data.frame()` etc) in anticipation of users trying that pattern first.

```r
is.tibble <- function(x) {
  deprecate_warn("2.0.0", "is.tibble()", "is_tibble()")

  is_tibble(x)
}
```

You can see this in the context of the source code [here](https://github.com/tidyverse/tibble/blob/dd29fb76b6b0335a08658c14483fe3cbb81cf7d7/R/tibble.R#L196). If a user calls `is.tibble()`, the responsibility is passed straight on to `is_tibble()` after a nudge towards the "preferred" tidyverse syntax.




## Multiple names for datasets?


Let's review again the key points we would like to achieve:

1. The package must contain only a single copy of the full dataset
2. The dataset must be accessible using at least two different names
3. Each name must work correctly with `data()`, i.e. `data("x", package = "pkg")` must load an object called `x` into the session


That third point is quite important for the usability of the package. In fact, if we spend some time thinking about the third point here, we actually get a big push in the right direction!

From the Details section of the documentation for `data()`:

> Currently, four formats of data files are supported:
>
>    1. files ending '.R' or '.r' are `source()`d in, with the R working directory changed temporarily to the directory containing the respective file. (`data` ensures that the **utils** package is attached, in case it had been run _via_ `utils::data.`)
>
>    2. files ending '.RData' or '.rda' are `load()`ed.
>    ...


We're familiar with the second format, `.rda`, but maybe we didn't know about the first! So it looks like we might be able to create a `.R` file which somehow loads the dataset which we have already saved elsewhere...

Let's make a file called `data/diamantes.R`, which - as we've just learned! - will be run using `source()` if our user calls `data("diamantes", package = "datapkg")`.

Now all we have to do is figure out how to "copy" our `diamonds` dataset into an object called `diamantes`.


Let's naively try a similar approach to the one we've seen used for functions. I'll add the following line to `data/diamantes.R`:

```r
diamantes <- diamonds
```

But attempting to build the package quickly results in a nasty error...

{% highlight rout %}
==> Rcmd.exe INSTALL --no-multiarch --with-keep.source datapkg

* installing to library 'C:/.../R/win-library/4.0'
* installing *source* package 'datapkg' ...
** using staged installation
** R
** data
*** moving datasets to lazyload DB
** byte-compile and prepare package for lazy loading
Error in eval(exprs[i], envir) : object 'diamonds' not found
Error: unable to load R code in package 'datapkg'
Execution halted
ERROR: lazy loading failed for package 'datapkg'
* removing 'C:/.../R/win-library/4.0/datapkg'
* restoring previous 'C:/.../R/win-library/4.0/datapkg'

Exited with status 1.
{% endhighlight %}

Note the `object 'diamonds' not found`!

This is because datasets aren't like other R objects: as [r-pkgs](https://r-pkgs.org/data.html#documenting-data) puts it,

> Objects in data/ are always effectively exported (**they use a slightly different mechanism than NAMESPACE but the details are not important**).

That "slightly different mechanism" is our problem - we can't simply refer to datasets as internal objects within our package! So we'll have to try a different approach...



## Let's read that again


Remember what the documentation for `data()` said? (We only looked at it a minute ago!)

We've started to take advantage of

> 1\. files ending '.R' or '.r' are `source()`d in, with the R working directory changed temporarily to the directory containing the respective file

already - and now

> 2\. files ending '.RData' or '.rda' are `load()`ed

is also going to be helpful, for getting hold of our already-saved data!

Let's edit `data/diamantes.R` so that it contains just the following line:

```r
load("diamonds.rda")
```

Note that we are using a relative path to `diamonds.rda` because, as informed by the docs, our working directory is now the directory where `data/diamantes.R` lives, i.e. `data/`!

Now our package _does_ build and install, but we do get an ominous warning in the output...

{% highlight rout %}
[...]
** data
*** moving datasets to lazyload DB
Warning: object 'diamonds' is created by more than one data call
** byte-compile and prepare package for lazy loading
[...]
{% endhighlight %}

... which makes sense. When we call `data("diamantes", package = "datapkg")`, we simply end up calling the same thing as `data("diamonds", package = "datapkg")`, so we end up with an object called `diamonds` in our session rather than `diamantes`. We're closer, but we're not there yet!


Maybe we can rearrange `data/diamantes.R` slightly to rename things for us...

```r
load("diamonds.rda")
diamantes <- diamonds
```

This seems very close. We get the same warning about `diamonds` when we rebuild the package, but now when we call `data("diamantes", package = "datapkg")` we _do_ get an object called `diamantes`... we just _also_ get one called `diamonds`! This is because when we source `data/diamantes.R`, we end up with two objects in the environment at the end of the script (`diamonds` and `diamantes`), both of which are then pulled into our R session.

So we can add one more line to fix this problem:

```r
load("diamonds.rda")
diamantes <- diamonds
rm(diamonds)
```

And now we have achieved our goal! No more warning message when we build, and both `diamonds` and `diamantes` work in the way we wanted.



## Now do it in one line

Three lines was too many for you?? Okay - we can replace the current contents of `data/diamantes.R` with:

```r
diamantes <- local(get(load("diamonds.rda")))
```

Working from the inside out:

* `load("diamonds.rda")` loads `diamonds` into our environment, and returns a vector of names of all loaded objects, i.e. `c("diamonds")`
* `get()` retrieves the object with a name of `"diamonds"` from the _current_ environment, since we didn't provide a different environment via the `pos` argument - i.e. we now have the actual data frame which the `diamonds` variable name refers to
* `local()` means all of that happened in its own little throwaway environment - so we end up with the data frame object which was returned from `get()`, and everything else is just thrown away, including the `diamonds` object which had been created by `load()`
* And finally, we assign that data frame into an object called `diamantes`, which is the only object we created in our main environment and therefore the only object which will be created when we call `data("diamantes", package = "datapkg")`



## Documentation

Of course, it's important to document our datasets - again we can refer to the relevant chapter of the ever-so-useful [r-pkgs](https://r-pkgs.org/data.html) for guidance! The documentation should live in `R/data.R`: we only have to write one set of documentation, and then we can use the `@rdname` Roxygen tag to include the other names!

```r
#' Prices of 50,000 round cut diamonds.
#'
#' A dataset containing the prices and other attributes of almost 54,000
#' diamonds.
#'
#' @format A data frame with 53940 rows and 10 variables:
#' \describe{
#'   \item{price}{price, in US dollars}
#'   \item{carat}{weight of the diamond, in carats}
#'   ...
#' }
#' @source \url{http://www.diamondse.info/}
"diamonds"

#' @rdname diamonds
"diamantes"
```



## Epilogue

So far I've noticed **two** important things to bear in mind when using this approach:

* We don't have to stop at two names - if we want to, we can keep making new `data/*.R` files which point back to the same `.rda` file! HOWEVER, we shouldn't have two _files_ with names which differ only in capitalisation, e.g. `diamantes.R` and `DiaMantes.R` - this is not possible on Windows, and not a great idea on other platforms! (In this particular case, we could have `diamantes.rda` with `diamonds.R` and `DiaMantes.R` instead)
* If you build your package using the `--resave-data` flag, any `data/*.R` files will be sourced and resaved as `.rda` files as part of the build process, rendering all your hard work useless![^7]

[^7]: And possibly resulting in two identically-named files, if you had e.g. `hello.rda` and `HeLLO.R`.


I'd be keen to hear whether anyone else has success with this method too, or if there are better ways to achieve the same thing! We seemed to make it through the CRAN submission process for {mangoTraining} with no problems at all; but I'm curious as to how reproducible this is. In particular I would love to know whether the CRAN builds use `--resave-data` or not, and whether that would scupper this method for packages where resaving takes us over the package size limit!

---
