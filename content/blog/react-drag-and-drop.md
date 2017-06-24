+++
date = "2017-06-24T11:14:35-05:00"
title = "React Drag and Drop"
description = "Implementing a simple generic order-able list using react the react-dnd library"
tags = [ "react", "react-dnd", "Javascript" ]
categories = [ "Development", "Javascript", "How To" ]
+++

I recently needed to implement an order-able list using react for a project.
Of course the first thing I did was look for drag and drop libraries to help.

After a bit of research I stumbled upon [React DnD](https://github.com/react-dnd/react-dnd).

This seemed like a great library to use because

* It embraced a lot of react / redux paradigms such as unidirectional data flow and declarative rendering
* It has replaceable back-end's to support HTML5 drag and drop or touch support drag and drop
* It is testable
* It had almost 6000 stars on their Github

React DnD does come with pre-made components.
So I decided to create my own generic order-able list component which I am going to show in this post.

Everything I explain and mention can be referenced in the example Github repository that pairs with this post here.

[Example React Orderable List](https://github.com/amast09/react-orderable-list)

I am going to use Facebook's popular (and awesome) [Create React App](https://github.com/facebookincubator/create-react-app) library to get up and running quickly.
You will need to have **Node >= 6** on your machine to use the tool.

```bash
npm install -g create-react-app

create-react-app react-orderable-list

cd react-orderable-list/

```

Now we will add the required React DnD libraries.

```bash
npm install --save react-dnd

npm install --save react-dnd-html5-backend
```

We are now ready to start developing. Simply run

```bash
npm start
```

And you will get a hot reloading react environment to develop against.

The first step we need to take to get the React DnD working with our React app is to add the DragDropContextProvider to our root React element with an HTML5Backend.
