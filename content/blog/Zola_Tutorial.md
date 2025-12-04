+++
title="Creating a Profile and Blog page using Zola"
date="2025-12-03"
template="blog.html"
authors=["Vjaylakshman K",]
+++
Hello people :D,

Finally after years of procrastination and laziness, the stars have aligned and I have a blog and profile page! All it took me is a morning to set this all up and get it up and running and yes I will acknowledge that I had to use AI for generating the CSS files because I am no good with that. Anyway this blog is supposed to be a very easy approach as to how can you also make a blog page for yourself!

## SSG (Static Site Generators)
First of all let's get started with what an SSG is, according to Mozilla docs

>A static site generator (SSG) is a software used to generate static websites. A static website is comprised of HTML, CSS, and JavaScript files. Most importantly static sites do not have server-side logic, so for any given URL, all users will receive the same content. Authors write content in any form accepted by the generator, such as Markdown, reStructuredText, HTML, (and sometimes even React, and so on), and the generator compiles them into a set of optimized static files that can be rendered by the browser.

In short this allows you to just focus on writing content using markdown or any required format and the SSG automatically converts it into the required HTML files. This allows for less headaches in formatting the HTML page and instead focus on writing content using Markdown, which is very easy!

There are a lot of Static Site Generators that are available out there, let me list some of them

- [Jekyll](https://www.jekyllrb.com) - One of the most famous Static Site Generators, written in ruby
- [Hugo](https://www.gohugo.io) - A SSG but written in Golang, makes it really fast
- [Zola](https://www.getzola.org/) - A SSG but written in Rust, which I have used to create this page!

Honestly, any SSG should get your job done, but I wanted to use something simple and since Zola was also written in rust, I wanted to try it out!

## Setting up Zola
The Documentation had already provided the ways in which Zola could be installed, and I was using a Mac in order to set it up, therefore I used brew to download Zola
```bash
brew install zola
```

**Note:** [Installation Page](https://www.getzola.org/documentation/getting-started/installation/)

Since I am hosting this on github pages, I created a repository with my username "vjay15.github.io" and another repository where the website files are hosted, you can have a gitignore, otherwise it could be left empty. Clone the repositories locally and cd to the directory (The present working directory should be the website file repository's folder), then
```bash
zola init #in case you have added LICENSE do -f
```
Which should result in Zola asking a few questions to initialise your page
```
Welcome to Zola!
Please answer a few questions to get started quickly.
Any choices made can be changed by modifying the `config.toml` file later.
> What is the URL of your site? (https://example.com): https://vjay15.github.io
> Do you want to enable Sass compilation? [Y/n]: n
> Do you want to enable syntax highlighting? [y/N]: y
> Do you want to build a search index of the content? [y/N]: y

Done! Your site was created in /Users/vjay/vjay15.github.io

Get started by moving into the directory and using the built-in server: `zola serve`
Visit https://www.getzola.org for the full documentation.
```
The URL of the site is the default URL of your website, SASS is a preprocessor that allows you to use advanced features like variables, nesting etc. If you want to enable it, since I am using vanilla CSS I have disabled it. Syntax highlighting is used for codeblocks and search index allows for easy indexing of your pages for SEO optimization.

## Working with Zola
After you have successfully set up Zola you will end up with the directory structure like this
```bash
$ tree                                                                                                                                                                        
.
├── LICENSE
├── config.toml
├── content
├── static
├── templates
└── themes

5 directories, 2 files
```
I am not going to be explaining in detail about what each of these directories and files do, because the Overview page of Zola docs does a better job at it so do check it out.

[Overview Page](https://www.getzola.org/documentation/getting-started/overview/)

Now that you've read that you might have got the basic idea behind how Zola helps us organise our content using Markdown and HTML. Now I will explain the rest in a short manner, I might be skipping a lot of details here so I again suggest you to go read the docs if you want in depth explanation.

All of the static assets like images, files and CSS will go under the static directory, you are free to organize them into whatever folder structure you want, I have created separate folder for files, images and styles under the static directory.

Let us go through how I structure my templates
- The base.html is the base template for all the pages, it contains the header, footer, Navigation bar, linking the style sheets etc. 
- The index.html is my home page, self explanatory
- The page.html is the default page structure for all the different pages I have in my website
- The section.html is the page which I use to list out pages in a section, in my case I have only blog section so it lists all the blogs
- The blog.html is the page which is the base structure for all the blogs

The page URLs are constructed based on the name of the markdown files, so for example if you take my about me page the markdown name is about.md, it is constructed like
```https://vjay15.github.io/about```. For sections it will be ```https://example.com/section/<md_filename>```. Therefore be careful in naming your md files. The page titles for the navbar are stored in config.toml
```
 # Navigation items for navbar
navbar_items = [
    { name = "About", url = "/about" },
    { name = "Portfolio", url = "/portfolio" },
    { name = "Blog", url = "/blog" },
    { name="Bucket List", url="/bucketlist"}
]
```
and is displayed by looping through the items, in the base.html file
```html
<nav>
    <div class="nav-container">
      <a href="{{ config.base_url }}" class="site-name">Vjaylakshman K</a>
      <ul>
        {% if config.extra.navbar_items %}
          {% for item in config.extra.navbar_items %}
            <li><a href="{{ get_url(path=item.url) }}">{{ item.name }}</a></li>
          {% endfor %}
        {% endif %}
      </ul>
    </div>
  </nav>
```
The get_url is a built in function provided by Zola to fetch the permalink of the pages. You can then choose your favourite AI or LLM tool to generate a minimalist CSS template for your website. Test it out and check if it all works and looks well :D

### Setting up Github Pages for Zola
In order to finally deploy it on Github pages, create a main.yml file under ```.github/workflows``` and add the following YAML code. This is under the assumption that the page is going to be deployed in another repository. Before that make sure the DEPLOY_TOKEN is added to the secrets of the repository (This is a personal access token, check how to make one here: [Link](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens))
```yaml
# On every push this script is executed
on:
  push:
    branches:
      - main

permissions:
  contents: read

name: Build and deploy GH Pages
jobs:
  build:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout source repository
        uses: actions/checkout@v4
      
      - name: Install Zola
        run: |
          curl -L https://github.com/getzola/zola/releases/download/v0.18.0/zola-v0.18.0-x86_64-unknown-linux-gnu.tar.gz | tar xz
          sudo mv zola /usr/local/bin
      
      - name: Build Zola site
        run: zola build
      
      - name: Deploy to vjay15.github.io repository
        uses: peaceiris/actions-gh-pages@v3
        with:
          personal_token: ${{ secrets.DEPLOY_TOKEN }}
          external_repository: Vjay15/vjay15.github.io
          publish_dir: ./public
          publish_branch: main
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
```

In case you are publishing this website to elsewhere I suggest you to check the documentation: [Link](https://www.getzola.org/documentation/deployment/github-pages/)

This will run and deploy the page automatically the moment you push code to the repository. With that we have now successfully built a static site using Zola from scratch!
Another feature of Zola just like other SSGs is that you can use a pre-existing theme and use it instead. I like to control how the page looks by myself and keep it minimalistic, therefore I created the structure and just made AI write the styles hehe. 

### Optional: Adding Giscus for comments
I have added Giscus to all the blog pages in my website so I can receive comments, therefore I will also add on how to add Giscus. Giscus is a comments system built using Github Discussions. The comments are all tracked in a separate repository that is created by you, so it's entirely under your control!

Here are the steps to add Giscus to your website:
- Create a repository, can be named anything but name it in a way you can remember it, the repository should be public
- Go to Giscus app [page](https://github.com/apps/giscus), install it in the new repository created by choosing the option Selected repositories -> The repository you have created.
- Enable the Discuss feature, [Reference](https://docs.github.com/en/github/administering-a-repository/managing-repository-settings/enabling-or-disabling-github-discussions-for-a-repository)

After this go to the Giscus [website](https://giscus.app/) and enter the information it asks for, if you don't want to read through it just scroll down and under repository enter your ```username/repositoryname```, it will show a green tick if your repository is configured properly. Scroll down copy the ```<script>``` code which looks like:
```js
<script src="https://giscus.app/client.js"
        data-repo="[ENTER REPO HERE]"
        data-repo-id="[ENTER REPO ID HERE]"
        data-category="[ENTER CATEGORY NAME HERE]"
        data-category-id="[ENTER CATEGORY ID HERE]"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
```
The values will be autofilled since you had already given the username/repositoryname. This copied code can be appended to the template for your blog, in my case it's in blog.html and voila! the comment box is there along with a reaction button :D

I hope you guys learnt something out of this, do make your own profile/blog page its fun to learn stuff and share it to people!

More random blogs coming soon so stay tuned! Have an epic day.

#### Link to the repository: [Link](https://github.com/Vjay15/MyWebsite_Zola_files)

