Blog about engineering and product at HackerEarth.
Visit <a
href="http://engineering.hackerearth.com">engineering.hackerearth.com</a>.

**Introduction:**

We write engineering blog posts whenever we build something cool, which is always. So every now and then we'll be writing engineering & news blog posts.

**Setup:**

Both Engineering blog and news blog have the same setup instructions. Please follow steps mentioned below to setup news blog locally:

1. Make sure you have access to respective repositories.

2.  Clone the repository once you have access:

``` git clone https://github.com/HackerEarth/engineering.hackerearth.com.git
```

3. As mentioned above we use Jekyll to power our news/engineering blog, install Jekyll using the following command:
```
sudo gem install jekyll
```
or
```
sudo apt install jekyll
```

4. If you are unable to install jekyll, install rouge first
```
$ sudo gem install rouge
```

5. You may need to install kramdown also.
```
$ sudo gem install kramdown
```

6. Start the server by executing:
```
$ jekyll serve # It will start the server on: http://127.0.0.1:4000
```
To start the server on different port execute:
```
$ jekyll serve --port=5001 # It will start the server on: http://127.0.0.1:5001
```

7. Server should be running now so setup is complete.

*****

**Q. How to write a new post or edit existing ones?**

Please follow the instructions mentioned below to get started with writing **news/engineering posts**.

1. I assume you have already cloned the respective repository, so navigate to that directory.

2. There are two main sub directories (`_posts` and `images`) that you will be altering generally while adding or editing posts.

3. If you do not know how to write markdown please learn: [[ http://markdown-guide.readthedocs.io/en/latest/basics.html | Markdown Basics ]]. This covers almost everything to write a beautiful news post.

4. All markdown content (posts) goes to `_posts` directory and images goes to `images`. //Please check out how other posts have been written//.

5. The name of markdown file starts with date. For example: For [[ http://news.hackerearth.com/2016/02/08/filter-candidates-easily-in-reports/ | this post ]] the filename is: `2016-02-08-filter-candidates-easily-in-reports.md`.

6. If you have already added your content to `_posts/` named as `2016-05-26-demo-news-post.md`, it will be accessible on: `http://127.0.0.1:4000/2016/05/26/demo-news-post/`.(No need to restart your server, just refresh your browser.)

Important Notes: 

1. **Please make sure your news post is successfully rendering on your local before pushing it to master.**

2. **Add as many images or/and videos possible to explain your content.**

3. **Proof read your content again, you can take help from your peers.**
