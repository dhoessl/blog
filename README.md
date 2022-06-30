# Static Blogpage

This role uses [hugo](https://gohugo.io/) to create a static blog page.
It is possible to create the page locally to check if posts are working out.

After testing and writing posts the static page will be deployed on a host using a nginx docker container

# Installation

Install Hugo and create a new hugo page in the files directory.
```
apt install -y hugo
cd files
hugo new site blog
```

Go back to the directory this file is in and clone a [theme](https://themes.gohugo.io/) of your choice into the files directory.

For Example the PaperMod Theme:
```
cd ..
git clone https://github.com/adityatelange/hugo-PaperMod.git files/blog/PaperMod
```
Now you have to add the theme to the config file:
```
theme:
  - 'PaperMod'
```
