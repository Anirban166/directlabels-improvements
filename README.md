## Coding Plan and Methods

I’ll serially go through my objectives based on the points laid out [here](https://github.com/rstats-gsoc/gsoc2021/wiki/directlabels-improvements#coding-project-directlabels-improvements):

1) Host the directlabels site via GitHub pages (thereby replacing the current version hosted on r-forge.r-project.org) and create/configure software for automatic generation of a new documentation website:

## File Structure
The site will be hosted on the 'gh-pages' branch, which is a replication of the current master branch supplemented with some files/folders (in the repository’s root) from the old svn [repository](https://r-forge.r-project.org/projects/directlabels/), which includes:
The doc directory (`directlabels/pkg/directlabels/tests/doc`) along with the datasets required to run the tests contained therein, and the files required to build the documentation (docs folder) and rest of the website.
This structure of the gh-pages branch will look like: (/root)
```
tdhock/directlabels/tree/gh-pages
├───> content from the current master branch
├───> files/folders from the svn repository:
│     ├───> tests/doc folder
│     ├───> data/prostate.Rdata file
│     ├───> docs folder
│     ├───> tex folder
│     └───> files within the www/ folder (excluding docs/)
```
As per the readme of the directlabels GitHub repo, the path passed onto `dldoc()` is `~/R/directlabels/pkg/directlabels`, which must be the source directory Toby sir was working on locally.  
Considering the above structure of files in the gh-pages branch, the directlabels source directory would be `~/directlabels` when locally accessed by a runner cloning the repository, so I’ll have to change [Line 78](https://github.com/tdhock/directlabels/blob/54ccbb95e0079649d350865f8c063adfc8fbbf0b/R/doc.R#L78) to `setwd(file.path("docs"))` instead of the current `setwd(file.path("..", "..", "www", "docs"))`. 
If I don’t follow the previously laid down directory structure, I’ll need to make the above modification within `dldoc()` to correspondingly set up the working directory in order to successfully re-compute the docs before configuring the workflow. 
