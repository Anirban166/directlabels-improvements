## Coding Plan

I’ll serially go through my objectives based on the points laid out [here](https://github.com/rstats-gsoc/gsoc2021/wiki/directlabels-improvements#coding-project-directlabels-improvements):

1) Host the directlabels site via GitHub pages (thereby replacing the current version hosted on r-forge.r-project.org) and create/configure software for automatic generation of a new documentation website:

### File Structure
The site will be hosted on the 'gh-pages' branch, which is a replication of the current master branch supplemented with some files/folders (in the repository’s root) from the old svn [repository](https://r-forge.r-project.org/projects/directlabels/), which includes:

The doc directory (`directlabels/pkg/directlabels/tests/doc`) along with the datasets required to run the tests contained therein, and the files required to build the documentation (docs folder) and rest of the website. <br>
This structure of the gh-pages branch will look like: (/root)
```
tdhock/directlabels/tree/gh-pages
├───> content from the current master branch
└───> files/folders from the svn repository:
      ├───> tests/doc folder
      ├───> data/prostate.Rdata file
      ├───> docs folder
      ├───> tex folder
      └───> files within the www/ folder (excluding docs/)
```
As per the readme of the directlabels GitHub repo, the path passed onto `dldoc()` is `~/R/directlabels/pkg/directlabels`, which must be the source directory Toby sir was working on locally.  

Considering the above structure of files in the gh-pages branch, the directlabels source directory would be `~/directlabels` when locally accessed by a runner cloning the repository, so I’ll have to change [Line 78](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/doc.R#L78) to `setwd(file.path("docs"))` instead of the current `setwd(file.path("..", "..", "www", "docs"))`. 

If I don’t follow the previously laid down directory structure, I’ll need to make the above modification within `dldoc()` to correspondingly set up the working directory in order to successfully re-compute the docs before configuring the workflows. 

### Workflows

With the help of GitHub Actions, the documentation site will be auto-generated at https://tdhock.github.io/directlabels/docs/index.html.

In order to achieve this, I will create two workflow files, the first of which will commit the changes to the R files (used to generate the `man` files which in turn update the documentation) made in the master branch (`directlabels/tree/master/R/*.R`) to the gh-pages branch. I’ll achieve this by using the [GitHub Pages Deploy Action](https://github.com/marketplace/actions/deploy-to-github-pages) ([alternative](https://github.com/crazy-max/ghaction-github-pages)), which will deploy changes to gh-pages (or any specified branch from where GitHub-pages is being built) when it is triggered on a push from another branch, which in our case would be the master branch.

But note that we only require the R files within the `R/` directory. Hence, I’ll specify a [path filter](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-including-paths) within the push field and use the `**` (include all) wildcard suffixed by `.R` to only include changes to the R files in `R/`:
```yaml
name: Commit changes to gh-pages
on:
  push:
    branches:
      - master
    paths:
      - R/**.R
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2.3.1
      - name: Deploy 
        uses: JamesIves/github-pages-deploy-action@4.1.0
        with:
          branch: gh-pages 
          folder: R
```          
With the changes now available in gh-pages, it’s time to re-compute the docs locally on the runner and then push back the changes to this branch.

The second workflow file will do this by first creating a local source directory of directlabels by cloning the gh-pages repository and by subsequently re-computing the docs/ therein. Finally, it’ll push back the changes (via use of Git) to our repository.

Since this will need to follow after our first workflow, I’ll conditionally run it based on the former workflow’s trigger under a [workflow_run](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#workflow_run) event:
```yaml
on:
  workflow_run:
    workflows: ["Commit Changes to gh-pages"]
    types:
      - completed
```
I can also use the `page_build` condition which runs a workflow after someone (in our case, it’ll be the GitHub Actions bot) pushes to a GitHub Pages-enabled branch, which triggers the `page_build` event. I’ll keep this as an option.

Under the `jobs` section, the first step I’ll specify will be as usual, the checkout action. But this time I’ll need to modify the token it uses since I’ll be pushing code in my workflow and using `git` commands. Hence, I’ll use a personal access token (`PAT`) instead of the default `GITHUB_TOKEN`:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          token: ${{secrets.PAT}}
      - name: ...
        run: ...
```        
Thereafter, I’ll follow up with the rest of the steps:
First, I’ll use the actions provided by [r-lib](https://github.com/r-lib/actions) to setup R on our runner:
```yaml
- name: Setup R 
  uses: r-lib/actions/setup-r@master
```
Next, I’ll install the required dependencies: (including ones required to run `dldoc()`)
```yaml
- name: Install dependencies
  run: |
  R -e 'install.packages(c("ggplot2", "directlabels", "inlinedocs", "reshape2", "mlmRev", "lars", "latticeExtra"))'
```  
Then I’ll clone directlabels and switch to the gh-pages branch: 
```yaml
- run: |
    cd ~
    git clone https://github.com/tdhock/directlabels
    cd directlabels
    git checkout gh-pages
```    
Finally, I’ll compute the documentation:
```yaml
- name: Compute the docs
  env: 
    repo_token: ${{ secrets.GITHUB_TOKEN }} 
  run: |
  R -e 'inlinedocs::package.skeleton.dx("~/directlabels")'
  # R -e 'source("R/dldoc.R")'
  R -e 'directlabels::dldoc("~/directlabels")'
```  
With the updated documentation files locally available in `~/directlabels` on the runner, we just need to add, commit and push back the changes to gh-pages: 
```yaml
- run: |
    git add .
    git commit -m "Updated documentation”
    git push
```    
Note that this wouldn’t have been possible without a personal access token. The checkout action I defined on the first step will store the passed account credentials by default so that subsequent git commands can pick them up automatically.

2) Adding directlabels to https://exts.ggplot2.tidyverse.org/gallery/:

This would be fairly straight-forward to accomplish given that I just need to follow the relatively simple instructions as mentioned [here](https://github.com/ggplot2-exts/gallery#adding-a-ggplot2-extension). I’ve drafted the basic key-value pair code segment to add to their `_config.yml` under the `widgets` section:
```yml
name: directlabels
thumbnail: images/image_name.png
url: http://tdhock.github.io/directlabels
jslibs: >
ghuser: tdhock
ghrepo: directlabels
tags: Direct-labelling, Positioning, Plotting, Visualization
cran: true
examples: http://tdhock.github.io/directlabels/examples.html
ghauthor: tdhock
short: >
   Easily add direct labels to plots!    
description: >
   Simple, uniform framework for adding direct labels to lattice or ggplot2 plots.
```
For the description field above, I’m using the project description line from the old svn [repository](https://r-forge.r-project.org/projects/directlabels/). A more elaborate version would be the one used as the inline documentation for the `direct.label` function: “Modern plotting packages like lattice and ggplot2 show automatic legends based on the variable specified for color, but these legends can be confusing if there are too many colors. Direct labels are a useful and clear alternative to a confusing legend in many common plots.” 

The url field can be changed to https://github.com/tdhock/directlabels i.e. the GitHub version (as some contributors did), but I guess that would be preferable only as a final resort if we had no other choices, as specified within the meta requirements. Since we have a detailed and secure (following deployment on gh-pages, as work done from the first point) website, I think it would suit better as the landing page. 
We can change all these details and finalize the image to display during the course of discussion that will follow prior to sending the pull request. 

On a side note, I noticed (while looking at the most recent [commit](https://github.com/ggplot2-exts/gallery/commit/329b6ff4f449d2ab12f560d06e8af0185a52869a)) that the main contributor manually updates the repo-metrics (stars, forks, issues and watchers) for each repository within the `github_meta.json` file, which is undesired and would show inaccurate results for most of the time. Having recently ventured into GitHub Actions, it instantly occurred to me - why not use them for automating the metrics? (will need to fetch data from the GitHub API within set intervals, just like badges do to show those metrics) As it turns out, there actually is an [issue](https://github.com/ggplot2-exts/gallery/issues/61) on this, but it hasn’t been catered to yet. This is off-topic with respect to our project and my main focus, but I think I will try to implement this after GSoC.

3) Refactor the codebase to make it eligible to exercise `grid.force()` by using the new (Rversion >= 3.0.0) grid hook methods (`makeContent()`, in place of `drawDetails.dlgrob` and `makeContext()`, wherever necessary) and by making changes to the functions affected by this transition:

The codebase and its underlying functions follow the S3 object-oriented system, bearing the `<generic function>.<object class>` naming convention (apparently visible, but one can cross-check via `pryr::otype(object)`) and using `UseMethod` (within parent function `direct.label`) to dispatch the corresponding class-specific method when called by the regular method name. 

This is crucial to keep in mind, since I'll need to cover changes for the class part before moving onto modify the methods.
The first change will be for the `dlgrob` function (which defines a grid grob class to draw direct labels), wherein instead of creating a new `grob` class with `grob()`, I'll create a `gTree` using `gTree()`:
```r
dlgrob <- function(data, method, debug = FALSE, axes2native = identity, ...)
{  ...
   gtree(data = data, method = method, debug = debug, axes2native = axes2native, cl = "dlgrobtree", name = name, ...) 
}
```
No doubt, this sole change would affect the places where `dlgrob()` is called (for e.g. [Line 44](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/ggplot2.R#L44) in `ggplot2.R` and [Line 136](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/lattice.R#L136) in `lattice.R`), but that's because I'll need to create and add relevant grobs as children of the `gTree` inside a `makeContent` method for this grob class. After all, its those low-level grobs which account for generating the output, and irrespective of what changes we make, we'll need to call those basic building blocks in the end. 

The currently used approach is that of generating this `dlgrob` class from the basic `grob`, and then drawing the desired output via `grid.*` functions. 
The modified approach to be implemented is to generate our desired output by directly using `*Grob` functions (pre-defined in `grid` just like `grid.*` ones, and same in terms of the output drawn by all means) instead, and in order to collectively assess them by `makeContent()`, I'll put the relevant grobs to be drawn in the `gTree` created within the above grob class through `makeContent.dlgrobtree`, so as to generate the same output as before when we call `grid.draw`. ([Line 138](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/lattice.R#L138) of lattice for e.g.) 

For the changes to be made, consider the code of the former `drawDetails.dlgrob` method:
```r
> library(grid)
> library(directlabels)
> getS3method("drawDetails", "dlgrob")
function (x, recording) 
{
    cm.data <- x$data
    cm.data$x <- convertX(unit(cm.data$x, "native"), "cm", valueOnly = TRUE)
    cm.data$y <- convertY(unit(cm.data$y, "native"), "cm", valueOnly = TRUE)
    cm.data$groups <- factor(cm.data$groups)
    levs <- unique(cm.data[, c("groups", "colour")])
    code <- as.character(levs$colour)
    names(code) <- as.character(levs$groups)
    cm.data <- ignore.na(cm.data)
    if (is.null(cm.data$label)) {
        cm.data$label <- cm.data$groups
    }
    cm.data <- apply.method(x$method, cm.data, debug = x$debug, 
        axes2native = x$axes2native)
    if (nrow(cm.data) == 0) 
        return()
    colour <- cm.data[["colour"]]
    cm.data$col <- if (is.null(colour)) {
        code[as.character(cm.data$groups)]
    }
    else {
        colour
    }
    defaults <- list(hjust = 0.5, vjust = 0.5, rot = 0)
    for (p in names(defaults)) {
        if (!p %in% names(cm.data)) 
            cm.data[, p] <- NA
        cm.data[is.na(cm.data[, p]), p] <- defaults[[p]]
    }
    cm.data <- unique(cm.data)
    gpargs <- c("cex", "alpha", "fontface", "fontfamily", "col")
    gp <- do.call(gpar, cm.data[names(cm.data) %in% gpargs])
    if (x$debug) {
        print(cm.data)
    }
    text.name <- paste0("directlabels.text.", x$name)
    with(cm.data, grid.text(label, x, y, hjust = hjust, vjust = vjust, 
        rot = rot, default.units = "cm", gp = gp, name = text.name))
}
<bytecode: 0x7f8a4b316d30>
<environment: namespace:directlabels>
```
In terms of replacement, everything would remain the same inside the function body except for the last part, where there is a call to `grid.text` (since this is common to all plots). This will get replaced by a call to `textGrob()` in order to generate a `textGrob` object for drawing the label text. I'll of course, replace the existing method name (following S3 nomenclature) `drawDetails.dlgrob` with `makeContent.dlgrobtree`, changing the generic function to the new grid hook `makeContent`, and the object class to `dlgrobtree`, as specified for the `cl` defined within `dlgrob` for our base grob.

Now, in terms of things to add this scope, I'll need to retrieve the other type of grobs, i.e. figuratively the ones creating the boxes/box-shapes. For reference, keep the `textGrob` in mind, which I'll come back to in a moment.<sup>1</sup>

Issue with this approach is that we will require both these types of grobs (one for the text and one for the box) together inside `makeContent` to assemble in our `gTree`. For this to happen, I'll need to receive the grob(s) that will generate the box(es) from the methods which create them, after devising a way to pass them from function to function. 
The way I'll achieve this is by attaching the required grob(s) to the `data.frame` object (which is passed around) via an attribute field. 

For an example, consider the [draw.polygons](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L473) method with its call to `grid.polygon`: (note that the ellipsis-alike `...` notation I'm using here inside the function body implies the usual code that is in between, left unchanged)
```r
draw.polygons <- function(d, ...) 
{  
   ...
   for(i in 1:nrow(d))with(d[i,], 
   {
   ...
    grid::grid.polygon(
      L$x, L$y,
      default.units = "cm",
      gp = grid::gpar(col = box.color, fill = colour),
      name = "directlabels.draw.polygon"
    )
  })
  d$colour <- d$text.color
  d
}
```
Apart from the switch to `polygonGrob()` instead of the call to `grid.polygon()`, I will create a separate attribute in the `data.frame` object to be returned which holds this grob. The pseudocode depicting the general idea for this would look like:
```r
draw.polygons <- function(d, ...) 
{  
   ...
   attr(d, 'shapeGrob') <- grid::polygonGrob(...)
   # or attributes(d)$shapeGrob <- grid::polygonGrob(...)
  ...
  d
}
```
Note that the reason I'm naming the attribute `shapeGrob` is because there are other variations in the shapes (`draw.rects` gives rectangles for e.g., so if we were to pick `polygonGrob`, it wouldn't make literal sense for that) and we need to pick a generalized term for use in `makeContent`.

Since there are multiple grobs generated for each categorical variable in the code above (with `(x, y)` positions being kept in a list, and assigned in a loop altogether to `polygonGrob(x, y, ...)`) within the loop above, I will make the modifications to collect the attributes within the loop itself during GSoC. Likewise, I'll make similar changes to other such positioning methods that currently make use of `grid.*` functions:

| Name and Link @utility.function.R | Current grid.* function(s) | *Grob function(s) to replace with |
|---|---|---|
| [far.from.others.borders](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L3) | `grid.points` | `pointsGrob` |
| [calc.boxes](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L321) | `grid.rect` | `rectGrob` |
| [draw.polygons](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L473) | `grid.polygon` | `polygonGrob` |  
| [draw.rects](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L510) | `grid.rect` | `rectGrob` |
| [project.onto.segments](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L885) | `grid.segments` | `segmentsGrob` |
| [empty.grid](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/utility.function.R#L1231) | `grid.points`, `grid.segments` | `pointsGrob`, `segmentsGrob` |

Apart from `draw.polygons` and `draw.rects`, the rest functions call the `grid.*` methods for debugging purposes only (`calc.boxes` for e.g. creates a rectangle to highlight the viewport area, which is a rectangular region). Following the transition, they wouldn't need to store the generated grobs. 

Coming back to `makeContent.dlgrobtree`, we can now access the above defined `shapeGrob` via the data frame contained in the `dlgrob` list object `x`. Also, we already have our `textGrob` from before<sup>1</sup>, so with both types of grobs available, I can now assign them as children to our `gTree`:
```r
makeContent.dlgrob(x, recording) 
{
   ...
   tg <- with(cm.data, textGrob(label, x, y, hjust = hjust, vjust = vjust, 
        rot = rot, default.units = "cm", gp = gp, name = text.name))
   sg <- attr(x$data, 'shapeGrob')
   setChildren(x, gList(tg, sg))
}
```
This modified `gTree` will be returned as the result of this method so that `grid.draw` can draw the generated content. The modifications thus made will allow us to use `grid.force()` now since it affects all grobs that have a `makeContent` method. 

4) Setup code coverage and then a testing framework, with tests based on the grid grobs which are exposed via `grid.force()`:

I’ll set up `codecov` via `covr` to generate the code coverage results on every commit we stage and push to directlabels. For the testing framework, I’ll set up `testthat` (if time permits, I’m also thinking to investigate `tinytest` as a possible alternative) and create some simple tests for individual grid graphical objects which become accessible after imposing `grid.force()`. 

It should be easy for me to set up these now since I’ve already used them before in [testComplexity](https://github.com/Anirban166/testComplexity). Also, I’ve specifically written about code coverage, unit testing and their automation in one of my blog posts which focuses on [Software Development in R](https://anirban166.github.io/Software-Development/).

For example, here’s how I’ll set up code coverage: (with reference to my aforementioned blog post)
I’ll first run `devtools::use_coverage(pkg = ".", type = c("codecov"))` within my local development scope. For the freshly generated `codecov.yml` file, I’ll add the following lines:
```yml
comment: false
language: R
sudo: false
cache: packages
after_success:
Rscript -e 'covr::codecov()'
```
I’ll add the last two lines on a GitHub Actions workflow<sup>2</sup> as well, so as to automate the process of generating the code coverage reports from codecov after every commit.

For the initial take, Toby sir will need to log into codecov using his GitHub account and for once, give it access to the directlabels repository which will generate a token for us to use (which should be copied).
Thereafter, we can run `covr::codecov(token = "TokenValue")` with our supplied token value, which will upload the code coverage results (same as measured by `covr::package_coverage()`) to codecov and subsequently provide access to the dashboard and various graphs to view directlabels’ coverage. Optionally, a badge can be added to the readme.

I’m also thinking of setting up CI to automatically run the tests on say, a pull request. I've recently stumbled upon it, but to use `r-ci` seems convenient, given it's 'ready-to-use' script:
```yml
name: CI
on:
  push:
  pull_request:
env:
  USE_BSPM: "true"
  _R_CHECK_FORCE_SUGGESTS_: "false"
jobs:
  ci:
    strategy:
      matrix:
        include:
          - {os: windows-latest}
          - {os: macOS-latest}
          - {os: ubuntu-latest}
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v2
      - name: Bootstrap
        run: |
          curl -OLs https://eddelbuettel.github.io/r-ci/run.sh
          chmod 0755 run.sh
          ./run.sh bootstrap 
      # Install dependencies including suggested packages:
      - name: Install Dependencies
        run: ./run.sh install_all
      - name: Run Tests
        run: ./run.sh run_tests 
```        
<sup>2</sup>The code coverage automation part can be included within this script itself: (in case I do not resort to make a separate one)
```yml
      - name: Coverage
        if: ${{ matrix.os == 'ubuntu-latest' }}
        run: ./run.sh coverage
```
