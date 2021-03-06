# FastPager

[![Build Status](https://travis-ci.org/fast-pager/fast-pager.svg?branch=master)](https://travis-ci.org/fast-pager/fast-pager)

Cached Paginator for Node.JS

The goal of FastPager is to introduce an easy way to paginate a long list of results into pages through a storage strategy to improve web page loads and server performance.

The common web developer writes code such that pagination is done on every every page load. However, this may be inefficient and prevents scalability as the same result is reproduced multiple times for different users. One way to tackle this is to put this into a cache. 

FastPager combines file/memory caching with pagination.

# Usage

There are two stages involved in FastPager: building and fetching. The building process retrieves each page's results and storage it. When requested, the fetching process may retrieve from the storage. It is recommended to run the building process in a background process separate from the process that serves the webpages. 

## Building pages

There are two key information needed by FastPager when building pages: the context and the retrieval function.

- The context represents the series of pages being retrieved: it is needed for storage so that it can be uniquely identified later.
- The retrieval function takes in a page number and performs the actual retrieval and pagination logic. It is repeatedly called with increments to the `page` parameter until an empty return is returned. A Promise is expected to be returned from this function.

The following shows an example of a script that builds pages for posts sorted by date in descending order. In this example, [Sequelize](http://docs.sequelizejs.com/) is used for the script to retrieve the list of posts to be displayed. The pages are stored as JSON in files in a system temporary directory.
  
    const os = require('os');
    const path = require('path');
    const models = require('../models');

    const pager = new FastPager(path.join(os.tmpdir(), 'fastpager'));
    const POSTS_PER_PAGE = 10;
    
    function buildLatestPostsPages() {
      return pager
        .context('latest-posts')
        .from((page) => {
          return models.Post
            .findAll({
              order: ['datePosted', 'desc'],
              offset: (page - 1) * POSTS_PER_PAGE,
              limit: POSTS_PER_PAGE
            });
        })
        .build();
    };
    
    buildLatestPostsPages()
      .then(() => {
        console.log('Build Complete!');
      });
      
The build method returns a Promise which can be used to do additional work once the pagination build process is done. You can write a timer / scheduler around the `buildLatestPostsPages()` that periodically invalidate and rebuild the cache. One example using setTimeout is as follows:

    const PROCESSING_INTERVAL = 300000; // every 5 mins
    const startProcessing = () => {
      buildLatestPostsPages()
        .then(() => {
          console.log('Build Complete!');
          setTimeout(startProcessing, PROCESSING_INTERVAL);
        });
    };
    
    startProcessing();
    
## Retrieving pages

Once the pages are built, the webpages can retrieve the result without having to perform any pagination or retrieval from database logic:

    const os = require('os');
    const path = require('path');

    const pager = new FastPager(path.join(os.tmpdir(), 'fastpager'));

    app.get('/posts', (req, res, next) => {
      let page = req.query.page || 1;
      page = Number(page);
      if (page < 0) {
        page = 1;
      }
      return pager
        .context('latest-posts')
        .at(page)
        .retrieve()
        .then((posts) => {
          res.json(posts);
        })
        .catch((err) => {
          console.error(err);
          res.status(400);
        });
    });

# License

Open-source under The MIT License.
