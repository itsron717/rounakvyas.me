+++
title = "Optimising Overleaf to GitHub Integration"
disqus_identifier = "overleaftogithub"
date = "2020-05-04"
description = "Automating LATEX"
tags = [
    "tech",
    "api"
]
+++

Automating LaTeX to pdf conversion using travis ci and some bash.

---

One of the main reasons why I started this blog was to write about my projects and some interesting things that come along the way. I guess it takes a pandemic to make me write.

A couple of weeks ago, I was reading a few articles on adding continuous integration and deployment to my projects. In the midst of it, I found [this](https://www.varstack.com/2018/01/11/An-interesting-discovery/) article in which the author talks about setting up a CI pipeline to automate building PDFs from LaTex projects on overleaf. He elaborates on the mistakes and discoveries made while exploring GitLab's [repository mirror](https://docs.gitlab.com/ee/user/project/repository/repository_mirroring.html) feature.

As someone who uses LaTex projects on overleaf to make a resume, I thought of automating the process as well but using GitHub and Travis CI. I had some experience with Travis CI as I used it for another project of mine, [xkcd](https://github.com/itsron717/XKCD) (a cli tool to get xkcd comics right from the terminal). The idea in the article above was to automate the extraction of the source code of the project from overleaf, and then build it in GitLab. I was not able to find anything similar to GitLab's repository mirror feature for GitHub which checks the source of the repository every hour. I relied on Travis `cronjobs` for this.

Now, I don't update my resume every hour but I do it once a month or so. My initial pipeline to build the resume and upload it on my website was something like this:

- Edit the `.tex` files on Overleaf and compile to `.pdf`.

- Download the `.pdf` and move to the source of my website(or upload it to the drive and generate the link).

- Commit the changes and deploy to GitHub pages.

As you can see, we can do better.

## Automating using GitHub and Travis CI

Similar to GitLab's feature, I thought of using a cronjob to run a build every week which compiles the `.tex` project into the pdf and pushes the changes back to `pdf/rounakvyas.pdf` based on the most recent commit. Overleaf syncs with GitHub seamlessly and also provides a Git URL to access the source code of the project. Awesome.

### Compiling `.tex` to `.pdf` locally

There are a lot of cli tools that can compile `.tex` documents to `.pdf` such as [Pandoc](https://pandoc.org/getting-started.html),[ TeXlive](https://www.tug.org/texlive/), [MacTeX](https://tug.org/mactex/), etc. There were some errors in my LaTeX project so they were not compiling properly with the help of the cli tools. The doc was compiling on overleaf and I had to look for something similar to that. Fortunately, I found [latex-online](https://github.com/aslushnikov/latex-online) on GitHub while searching for more cli tools. It is an online latex compiler specifically for compiling `.tex` to `.pdf`. Perfect. I used its command-line interface, [laton](https://github.com/aslushnikov/latex-online#command-line-interface) to compile the document locally.

### Script to Update the Resume

Now that I was able to compile the documents. I needed a way to update the resume during each build, i.e, delete the older version and move the updated resume in `pdf/rounakvyas.pdf`. A simple bash script did the job.

**`./scripts/generate_pdf.sh`**

```bash
#!/bin/bash

generate(){
    # Check if file exists
    if [ -f pdf/rounakvyas.pdf ]; then
        $(rm pdf/rounakvyas.pdf)
        echo "Removing previous resume."
    fi
    #Compile doc and i/o redirection (stdout and stderr) to dev/null.
    $(./laton cv_7.tex > /dev/null 2>&1)
    $(sleep 5) # wait to generate .pdf
    $(mv cv_7.pdf pdf/rounakvyas.pdf)
}
```

### Push to GitHub from Travis CI

This was the most irritating part. The script to compile the document and move it to `pdf/rounakvyas.pdf` was ready. I just had to make Travis CI automatically push the changes to the master branch.

Travis supports `after_sucess`, a hook that is called when the build succeeds. I replaced the `deploy` section in `.travis.yml` with it.

Another bash script was required to add the changes to GitHub.

`.travis/push.sh`

```bash
#!/bin/sh
# credits - https://gist.github.com/willprice/e07efd73fb7f13f917ea

setup_git() {
  git config --global user.email "<enter_username>"
  git config --global user.name "<enter_email>"
}

commit_pdf_file() {
  git checkout master
  git add .
  git commit --message "Travis build: $TRAVIS_BUILD_NUMBER"
}

upload_file() {
  git remote rm origin
  git remote add origin https://itsron717:${GH_TOKEN}@github.com/itsron717/rounakvyas-tex.git > /dev/null 2>&1
  git push origin master --quiet > /dev/null 2>&1
}

setup_git
commit_pdf_file

if [ $? -eq 0 ]; then
  echo "Uploading updated resume to GitHub..."
  upload_file
else
  echo "No changes in resume."
fi
```

Now, a lot of things are happening in this piece of code. Firstly, One can push to GitHub automatically using a [Github deploy key](http://markbucciarelli.com/posts/2019-01-26_how-to-push-to-github-from-travis-ci.html) or a [personal access token](https://gist.github.com/willprice/e07efd73fb7f13f917ea). It was easier to encrypt a personal token on Travis so I went with that.

For this, one must generate a personal access token on GitHub. This secret token needs to be stored somewhere but cannot be stored directly in `.travis.yml`. If you’re using Travis on public GitHub repositories, your build log is publicly visible. If there are any Git related errors, it is possible that the origin URL (with your GitHub personal access token with access to ALL your public repositories) may be logged, which is a huge security risk.

Travis supports encrypted environment variables and can be added using Travis Command Line Client. Once this is done, `.travis.yml` stores the encrypted GitHub token. Now, that we have the access token available on Travis, we can write the script(above) to push the changes to GitHub.

A detailed explanation of this process can be found [here.](https://gist.github.com/willprice/e07efd73fb7f13f917ea)

## Conclusion

The final `.travis.yml` file looked something like this:

```yaml
language: shell
sudo: required
before_install:
  - chmod +x scripts/generate_pdf.sh
  - chmod +x .travis/push.sh
  - curl -L https://raw.githubusercontent.com/aslushnikov/latex-online/master/util/latexonline
    > laton && chmod 755 laton
script:
  - scripts/generate_pdf.sh
after_success:
  - if [ "$TRAVIS_EVENT_TYPE" = "cron" ]; then sh .travis/push.sh; else echo "Not cron. Skipping push to GitHub"; fi
env:
  global:
    secure: dkzLlvzJWjGG6rS9qGl3BFBxN3Il/L2s/hYeHPLW64Sj2L9xdQK3l/mEGqonLuPhw4tfNeByuo2FlYbArsKw9b51IHcrCfuHd/kuExfZuNo5j7ed4W5Zv8O4paqm9dur1/3ge2uCMzm2ueYaPmu+pv/1S2DS+hYEWuLzRzuwyg2AWtVCvRSwMJhmELmvA7f+zyRj9jsIMJaCq40wRZ/LKr7LZjAGflECedZE0s8GG+4YjO5cnxPaD33/i8ZS6d20wG1aMGY7FlzgDrEPN5+cZC0nNRKYTwfUZHdzZPZUx7gUw4q/9dgljmzATMnPDEzuF/MBhbK/V/I01cf9B3N1EMT7f88HixgRIdV/vElqNT+mWIEZHEowO3DEQUIKo5Ibcu08nP21QCWwmB7gYms1uSUSZMTrFjzURoTHe19X+2drAeA2HBZ/6zPXL7m/mY/+R5JXwH5nzlA8a/oDRgz46Ea7+W+5bxuJW6VaNeMMOQfolksLmcb1HxyEgSvFyFSUQSMTl6vp/CxqqYuhXiJZuA2EY27olZ2pxegmIerQb8EoEShf91b5laGKRU4Hj8Pc1F7BqXJ7hI9l4m6P1lu7aEw2j5spjZummzLyr8hPrk3mHCfXE1Txy7NkGZYaTShtKLEp9aHZ6ewVBDX7T6fmAt1keJvZFmpyOCEn6Hydd6U=
```

Now, all I have to do is edit the document on overleaf, push to GitHub and the `.pdf` are generated and updated every week automatically!

Here is the [link](https://github.com/itsron717/rounakvyas-tex) to the GitHub repository.

## To-Do

Currently, the `pdf` is being generated in the same repo as the `.tex` project. I can push the `pdf` changes directly to GitHub Pages but that would require me to rewrite the CI for my website. I'll do this sometime, maybe.

Anyway, that was it. I spent an entire day to automate 4-5 minutes of every month. I guess this is how it is during the quarantine.

## References

- Will Price’s [GitHub Gist](https://gist.github.com/willprice/e07efd73fb7f13f917ea)

- Vinay Gopinath's [post](https://www.vinaygopinath.me/blog/tech/commit-to-master-branch-on-github-using-travis-ci/)

## Contact Me

If something was not clear or right in the above post or if something could have been done in a much more efficient manner then do let me know. :)
