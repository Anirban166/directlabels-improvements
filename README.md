## Coding Plan and Methods

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

Todo

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
I’ll add the last two lines on a GitHub Actions workflow1 as well, so as to automate the process of generating the code coverage reports from codecov after every commit.

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
The code coverage automation part can be included within this script itself: (in case I do not resort to make a separate one)
```yml
      - name: Coverage
        if: ${{ matrix.os == 'ubuntu-latest' }}
        run: ./run.sh coverage
```
