# Development log

All of the steps I'm taking to build this app, from start to finish.

### Day 1: Concept and tech

It's Friday 5/13, and we have decided to come up with a new example app.

[We wrote down all of the stuff we want to demonstrate about building an app with Apollo.](README.md)

I've also made a list of technology choices, listed in the README:

- Apollo server - to put a nice unified API on top of our GitHub and local data
- Apollo client - to load that data declaratively into our UI
- React - it's a great way to build UIs, and has the best integration with Apollo and Redux
- React router - it seems to be the most popular React router today, and has some great hooks and techniques for SSR. It seems like James Baxley has had some success with implementing this stuff already with React Router.
- Webpack - the Meteor build system is by far the most convenient, but comes with a dependency on mongo, its own account system, etc. Since we want to learn how Apollo works without all of these things, we're going to not use it even though it would reduce the complexity of the code.
- Babel - to compile our server code.
- Redux - to manage client side data, perhaps we can also use Redux Form to manage the submission form, but we'll see when we get there.
- Passport.js for login - this seems to be the most common login solution for Express/Node, and has a great GitHub login solution. Jonas has already written a tutorial for using that with GraphQL.
- SQL for local data - We'll use SQLite for dev, and Postgres for production. TBD - should we use Sequelize or some other ORM? Or just Knex for query building?
- Bootstrap for styling - I already know how to use it, and I don't want the styling to be the focus of the app. Let's keep it as simple as possible.

I drew a [quick mockup of the desired UI](mockup.jpg). It's quite simple, and should be easy to implement in bootstrap.

### Day 2: Listing different views and designing schema

We want to make this app as simple as possible, but also useful. I think I'll design it the way I would a real app, and then simplify if necessary. Here are some ideas for different views in the app:

- Home page: "Hot" feed, which ranks by number of upvotes and time since posting
- "new" page, which lists stuff that was just posted, but hasn't had time yet to be upvoted
- Submission page, which is a form that lets people put in info about the repository - perhaps it's just a link to the repo, maybe a comment as well?
- Repo page, where you can look at a particular repository, and its upvotes and comments

Given this, let's design our GraphQL schema for the app. I think this is a great place to start the process because it will let us set up some mocked data, and work on the UI and API in parallel as we need to.

#### Repository

This represents a repository on GitHub - all of its data is fetched from the GitHub API. I think it will be convenient to separate data from different APIs into different types in my schema, rather than having one type that merges both the local data (upvotes etc) and the GitHub data (description etc). This can theoretically have [all of the fields that GitHub returns](https://developer.github.com/v3/repos/#get), but we're probably interested mostly in high-level data like:

- Repository name
- Organization/author avatar
- Description
- Number of stars
- Number of issues
- Number of PRs
- Number of contributors
- Date created

#### Entry

This represents a GitHub repository submitted to GitHunt. It will have data specific to our application:

- User that posted this repository
- Date posted
- The related repository object from GitHub
- Display score (probably just upvotes - downvotes)
- Comments
- Number of comments

#### Comment

Just a comment posted on an entry.

- User that posted this comment
- Date posted
- Comment content

#### User

Someone that has logged in to the app.

- Username
- Avatar
- GitHub profile URL

#### In GraphQL schema language

Now let's put it all together to see what our GraphQL schema might look like. Let's keep in mind that this is just a sketch - the mechanics of our app might require some changes.

```graphql
# This uses the exact field names returned by the GitHub API for simplicity
type Repository {
  name: String!
  full_name: String!
  description: String
  html_url: String!
  stargazers_count: Number!
  open_issues_count: Number

  # We should investigate how best to represent dates
  created_at: String!
}

# Uses exact field names from GitHub for simplicity
type User {
  login
  avatar_url
  html_url
}

type Comment {
  postedBy: User!
  createdAt: String! # Actually a date
  content: String!
}

type Entry {
  repository: Repository!
  postedBy: User!
  createdAt: String! # Actually a date
  score: Number!
  comments: [Comment]! # Should this be paginated?
  commentCount: Number!
}
```

Looks pretty good so far! We might also want some root queries as entry points into our data. These are probably going to correlate to the different views we want for our data:

```graphql
# To select the sort order of the feed
enum FeedType {
  HOT
  NEW
}

type Query {
  # For the home page, after arg is optional to get a new page of the feed
  # Pagination TBD - what's the easiest way to have the client handle this?
  feed(type: FeedType!, after: String): [Entry]

  # For the entry page
  entry(repoFullName: String!): Entry

  # To display the current user on the submission page, and the navbar
  currentUser: User
}
```