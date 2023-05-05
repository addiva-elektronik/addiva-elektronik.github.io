Tech Blog and Course Material
=============================

We use [Hugo](https://gohugo.io) to generate: <https://addiva-elektronik.github.io/>

To contribute to the blog, either fork this repository, or if you are a
registered employee, clone the repository:

    cd
    git clone git@github.com:addiva-elektronik/addiva-elektronik.github.io.git blog
	cd blog/

Make changes/additions on a separate branch:

    git checkout -b my-changes

When you push it to GitHub you will get a question by GitHub.com if you
want to create a pull request.  Do that and follow the instructions.

To add/modify blog posts:

 1. [Install Hugo](https://gohugo.io/installation/): `sudo apt install hugo`
 2. New blog posts go in `content/posts/yyyy-mm-dd-my-first-post.md`
 3. Add front matter at the top of your post
 
        ---
        title: "My First Post"
        date: 2022-11-20T09:03:20-08:00
        draft: true
        ---

 4. [Add content ...](https://gohugo.io/getting-started/quick-start/#add-content)
 5. Add tags and categories where applicable
 
The preview your post/changes:

 - `cd /path/to/top-directory-of-blog/`
 - Run `hugo server`, follow the instructions

When you are happy, commit the changes to your branch and create a pull request.
