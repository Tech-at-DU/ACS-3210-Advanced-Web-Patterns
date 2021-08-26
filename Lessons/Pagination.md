# 📄🔍 Day 1: Pagination

<!-- > -->

<!-- omit in toc -->
## ⏱ Agenda {docsify-ignore}

1. [🏆 Learning Outcomes](#%F0%9F%8F%86-learning-outcomes)
1. [👋 Welcome to Class](#%F0%9F%91%8B-welcome-to-class)
1. [📁 Intro Project](#%F0%9F%93%81-intro-project)
   1. [Project Overview](#project-overview)
1. [Existing Vs New Projects](#existing-vs-new-projects)
1. [Pagination](#pagination)
   1. [Why do we need pagination?](#why-do-we-need-pagination%3F)
   1. [Question](#question)
1. [[**10m**] 🌴 BREAK {docsify-ignore}](#%5B%2a%2a10m%2a%2a%5D-%F0%9F%8C%B4-break-%7Bdocsify-ignore%7D)
1. [Activity: Technical Debate - Picking a Pagination Module](#activity%3A-technical-debate---picking-a-pagination-module)

<!-- > -->

## 🏆 Learning Outcomes

By the end of this lesson, you should be able to...

- Understand how pagination works, and how to implement it

<!-- > -->

## 👋 Welcome to Class

Instructor will walk through the [syllabus](https://make.sc/bew2.1) and answer questions about the course.

**Students**, remember to join the following:

1. Course Slack channel, `##bew-2-1-web-patterns`!
1. Gradescope: **INSERT LINK HERE**

<!-- > -->

## 📁 Intro Project

Wait a minute, a project _already?!_

**How Not To React:**

![fire_spongebob](assets/fire_spongebob.gif)

<!-- v -->

**Instead:**

![do_it](assets/do_it.gif)

<!-- v -->

### Project Overview

**TODO:**

- Add project description
- Where code will be written


<!-- > -->

## Existing Vs New Projects

You may be asking, "why are we enhancing an existing project instead of creating a new one?"

![teams](assets/teams.jpg)

- More common as a junior engineer to be working on an existing project
- You will have to revamp/add features to an existing project at some point in your career
- Improve your existing portfolio projects to make them stand out more


<!-- > -->

## Pagination

### Why do we need pagination?

**Question:** What is the fastest way to speed up a query for 1,000,000,000 (1 Billion) records?

<!-- v -->

<details>
  <summary>
    The answer (do not click!)
  </summary>
  Use pagination to only return the first 20 records like .... Google does!
  <img src='google.png' />
</details>

<!-- v -->

![pagination](assets/pagination.png)

If you look around almost every website is paginated. Why? Probably because pagination is **one of the easiest ways to speed up page loads**. If you are loading 1000 records on your index page, that will take 10 seconds to load. Pagination will speed it up by sending only the first 20 records.

<!-- v -->

### Question

What are other benefits to pagination?

<!-- v -->

- Improved structure and readability: reduced chance of users getting lost
- Separate URLs for pages for ease of reference
- Positive effect on SEO, easier for crawlers to navigate

<!-- > -->


## [**10m**] 🌴 BREAK {docsify-ignore}

<!-- > -->


## Activity: Technical Debate - Picking a Pagination Module

![debate](assets/debate.jpeg)

Throughout your career, you will have many technical debates with your teams. We're going to practice that here:

1. Compare and contrast these modules, list their pros and cons, and decide which one you would use and why.

     - [paginate](https://www.npmjs.com/package/paginate)
     - [mongoose-paginate](https://www.npmjs.com/package/mongoose-paginate)
     - [express-paginate](https://www.npmjs.com/package/express-paginate)
2. Divide into groups where everyone in the group agrees on which package they would use
3. Now split your group into thirds, one third stay stays, the other two thirds go to another group and try to convince them to use your module.
4. Could you convince anyone to change? What arguments are the most compelling for people?  What arguments were most compelling to you?

<!-- > -->

