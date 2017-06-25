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

React DnD does not come with pre-made components.
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

Now we will add the required React DnD libraries and other dependencies.

```bash
npm install --save react-dnd

npm install --save react-dnd-html5-backend

npm install --save lodash

npm install --save prop-types
```

We are now ready to start developing. Simply run

```bash
npm start
```

And you will get a hot reloading react environment to develop against.

The goals for our orderable list is

* Keep it as simple as possible
* The user of the component should only need to worry about the list items themselves as well as what the list is wrapped in
* Provide a hook for the consumer of the component to know when a list item has been moved and where it was moved to
* Allow for the user to specify a drag element if they don't want the entire list item to be used as a handle

The first step we need to take to get the React DnD working with our React app is to add the DragDropContextProvider to our root React element with an HTML5Backend.

[DragDropContextProvider](http://react-dnd.github.io/react-dnd/docs-drag-drop-context-provider.html)

```jsx
<DragDropContextProvider backend={HTML5Backend}>
    <div className="App">
      <div className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <h2>Welcome to React</h2>
      </div>
      <p className="App-intro">
        To get started, edit <code>src/App.js</code> and save to reload.
      </p>
    </div>
  </DragDropContextProvider>
```


Now we will create our Draggable component for our orderable list,

`DraggableItem.jsx`

```jsx


import React, { Component } from 'react';
import PropTypes from 'prop-types';
import _ from 'lodash'
import { DragSource, DropTarget } from 'react-dnd';

const itemSource = {

  beginDrag (props) {
    return {
      id: props.id,
      originalIndex: props.findItem(props.id).index,
    };
  },

  endDrag (props, monitor) {
    const { id: droppedId, originalIndex } = monitor.getItem();

    if (!monitor.didDrop()) {
      props.moveItem(droppedId, originalIndex);
    } else {
      props.itemMoved({ id: droppedId, droppedIndex: props.findItem(droppedId).index });
    }
  }

};

const dropTargetCollect = (connect) => ({
  connectDropTarget: connect.dropTarget()
});

const itemTarget = {

  canDrop () {
    return false;
  },

  hover (props, monitor) {
    const { id: draggedId } = monitor.getItem();
    const { id: overId } = props;

    if (draggedId !== overId) {
      const { index: overIndex } = props.findItem(overId);
      props.moveItem(draggedId, overIndex);
    }
  },

};

const dragSourceCollect = (connect, monitor) => ({
  connectDragSource: connect.dragSource(),
  connectDragPreview: connect.dragPreview(),
  isDragging: monitor.isDragging()
});

class DraggableItem extends Component {

  render() {
    const { element, isDragging, connectDragSource, connectDropTarget, connectDragPreview, id } = this.props;
    const opacity = isDragging ? 0 : 1;
    let item;

    if (element.handleElementIndex !== undefined) {
      element.children[element.handleElementIndex] = connectDragSource(element.children[element.handleElementIndex]);
      item = connectDragPreview(connectDropTarget(<element.parentWrapperTag style={{ opacity }} key={id}>{element.children}</element.parentWrapperTag>));
    } else {
      item = connectDragSource(connectDropTarget(<element.parentWrapperTag style={{ opacity }} key={id}>{element.children}</element.parentWrapperTag>));
    }

    return item;
  }

}

DraggableItem.propTypes = {
  connectDropTarget: PropTypes.func.isRequired,
  connectDragSource: PropTypes.func.isRequired,
  connectDragPreview: PropTypes.func,
  isDragging: PropTypes.bool.isRequired,
  id: PropTypes.any.isRequired,
  element: PropTypes.shape({
    children: PropTypes.arrayOf(PropTypes.element),
    parentWrapperTag: PropTypes.string.isRequired,
    handleElementIndex: PropTypes.number
  }).isRequired,
  moveItem: PropTypes.func.isRequired,
  findItem: PropTypes.func.isRequired,
  itemMoved: PropTypes.func.isRequired
};

export default _.flow(
  DragSource('Item', itemSource, dragSourceCollect),
  DropTarget('Item', itemTarget, dropTargetCollect)
)(DraggableItem);


```

This component is both a DragSource (we can drag it) and a DropTarget (we can drop other draggable elements on to it).


Here is React DnD's documentation for [DragSource](http://react-dnd.github.io/react-dnd/docs-drag-source.html)

The extra logic we added is in the `endDrag` function

```jsx
endDrag (props, monitor) {
    const { id: droppedId, originalIndex } = monitor.getItem();

    if (!monitor.didDrop()) {
      props.moveItem(droppedId, originalIndex);
    } else {
      props.itemMoved({ id: droppedId, droppedIndex: props.findItem(droppedId).index });
    }
}
```

When the element stops being dragged, if the item was not dropped then we want to move it back to it's original position (Cancelling the drop if it isn't on another item).

If it was dropped, propagate that event into the hook we provided to the user of this component.


Here is React DnD's documentation for [DropTarget](http://react-dnd.github.io/react-dnd/docs-drop-target.html)

```jsx
const itemTarget = {

  canDrop () {
    return false;
  },

  hover (props, monitor) {
    const { id: draggedId } = monitor.getItem();
    const { id: overId } = props;

    if (draggedId !== overId) {
      const { index: overIndex } = props.findItem(overId);
      props.moveItem(draggedId, overIndex);
    }
  },

};

```

We aren't allowing the target to be dropped on `canDrop () { return false; },` because instead of being able to be dropped on we are reordering the elements.

The `hover (props, monitor) {` does the re-ordering of the elements when one element is hovered over another.

Our render function is the following,

```jsx
render() {
  const { element, isDragging, connectDragSource, connectDropTarget, connectDragPreview, id } = this.props;
  const opacity = isDragging ? 0 : 1;
  let item;

  if (element.handleElementIndex !== undefined) {
    element.children[element.handleElementIndex] = connectDragSource(element.children[element.handleElementIndex]);
    item = connectDragPreview(connectDropTarget(<element.parentWrapperTag style={{ opacity }} key={id}>{element.children}</element.parentWrapperTag>));
  } else {
    item = connectDragSource(connectDropTarget(<element.parentWrapperTag style={{ opacity }} key={id}>{element.children}</element.parentWrapperTag>));
  }

  return item;
}
```

Unfortunately we can't allow the user of the OrderableList to pass just the list element itself in because they may want to specify a handle.
If we want a handle we need to wrap the element in a connectDragSource and the entire element in a connectDragPreview.
Which would be impossible to do if they passed in only a single element. Lastly we wrap the element itself in the tag they specified.


The last thing of note is the export command,

```jsx
export default _.flow(
  DragSource('Item', itemSource, dragSourceCollect),
  DropTarget('Item', itemTarget, dropTargetCollect)
)(DraggableItem);

```

It wires together our draggable component with the DragSource and DragTarget code we configured.

Now we will create our Orderable List component,

`OrderableList.jsx`

```jsx

import React, { Component } from 'react';
import PropTypes from 'prop-types';
import update from 'react/lib/update';
import { DropTarget } from 'react-dnd';
import DraggableItem from './DraggableItem';
import _ from 'lodash';

const itemTarget = {
  drop() {},
};

const dropTargetCollect = (connect) => ({
  connectDropTarget: connect.dropTarget()
});


class OrderableList extends Component {

  constructor(props) {
    super(props);
    this.moveItem = this.moveItem.bind(this);
    this.findItem = this.findItem.bind(this);
    this.state = { items: this.props.items };
  }

  componentWillReceiveProps(nextProps) {
    this.setState({ items: nextProps.items });
  }

  moveItem(id, atIndex) {
    const { item, index } = this.findItem(id);
    this.setState(update(this.state, {
      items: {
        $splice: [
          [index, 1],
          [atIndex, 0, item],
        ],
      },
    }));
  }

  findItem(id) {
    const { items } = this.state;
    const item = _.find(items, c => c.id === id) || {};

    return {
      item,
      index: items.indexOf(item),
    };
  }

  render() {
    const { connectDropTarget, dropHandler } = this.props;
    const { items } = this.state;

    return connectDropTarget(
      <this.props.containingTag>
        {items.map(item => (
          <DraggableItem
            key={item.id}
            id={item.id}
            element={item.element}
            moveItem={this.moveItem}
            findItem={this.findItem}
            itemMoved={dropHandler || _.noop}
          />
        ))}
      </this.props.containingTag>,
    );
  }
}

OrderableList.propTypes = {
  connectDropTarget: PropTypes.func.isRequired,
  items: PropTypes.arrayOf(
    PropTypes.shape({
      id: PropTypes.number,
      element: PropTypes.shape({
        children: PropTypes.arrayOf(PropTypes.element),
        parentWrapperTag: PropTypes.string.isRequired,
        handleElementIndex: PropTypes.number
      }).isRequired,
    })
  ).isRequired,
  dropHandler: PropTypes.func,
  containingTag: PropTypes.string.isRequired
};

export default DropTarget('Item', itemTarget, dropTargetCollect)(OrderableList);
```

This component is only a DropTarget (we can drop other draggable elements on to it).


Here is reference to React DnD's documentation again for [DropTarget](http://react-dnd.github.io/react-dnd/docs-drop-target.html)


```jsx
componentWillReceiveProps(nextProps) {
  this.setState({ items: nextProps.items });
}
```

We want to update our component anytime our parent passes us new/updated items (our items may be coming in asynchronously).

```jsx
moveItem(id, atIndex) {
    const { item, index } = this.findItem(id);
    this.setState(update(this.state, {
      items: {
        $splice: [
          [index, 1],
          [atIndex, 0, item],
        ],
      },
    }));
}
```

Simple function to move the item to the specified index.

```jsx
findItem(id) {
    const { items } = this.state;
    const item = _.find(items, c => c.id === id) || {};

    return {
      item,
      index: items.indexOf(item),
    };
}
```

Simple function to find the item in the list based on the id.

Our `render() {` function just iterates over all our draggable items rendering them. It also wraps the component in the specified tag our orderable list specified.

The last thing that we do is wrap our element into a DropTarget

```jsx
export default DropTarget('Item', itemTarget, dropTargetCollect)(OrderableList);
```

Now that we have our generic DraggableItem and OrderableList of Draggable items we can finally utilize them.

We want to create the element that will be draggable first. A simple movie item to be able to rank our favorite movies.

`MovieItem.jsx`

```jsx
import React from 'react';
import PropTypes from 'prop-types';

const MovieItem = props => (
  <div key={`movie-item__${props.id}`}>
    <span key={`movie-item__${props.id}-title`}>{props.title}</span>
    <span key={`movie-item__${props.id}-genre`}>{props.genre}</span>
    <span key={`movie-item__${props.id}-year`}>{props.year}</span>
  </div>
);

MovieItem.propTypes = {
  id: PropTypes.number.isRequired,
  title: PropTypes.string.isRequired,
  year: PropTypes.string.isRequired,
  genre: PropTypes.string.isRequired
};

export default MovieItem
```

This is a simple visual component just displaying data with no logic.

Now we will make our customized list component.

`MovieItemList.jsx`

```jsx

import React from 'react';
import _ from 'lodash';
import OrderableList from './OrderableList';
import MovieItem from './MovieItem';

class MovieItemList extends React.Component {

  constructor(props) {
    super(props);
    this.state = { movies: [] };
  }

  movieRankUpdate (reRankedMovie) {
    console.log(reRankedMovie);
  }

  mapMovieToDraggableMovieElement (movie) {
    return {
      id: movie.id,
      element: {
        children: [
          <MovieItem key={`movie-${movie.id}__container`} movie={movie}/>,
          <span key={`movie-${movie.id}__icon`}>{" <>"}</span>
        ],
        parentWrapperTag: 'div',
        handleElementIndex: 1
      }
    };
  };

  componentDidMount () {

    const movies = [
      {
        id: 1,
        title: 'Step Brothers',
        year: '2008',
        genre: 'Comedy'
      },
      {
        id: 2,
        title: 'Elf',
        year: '2003',
        genre: 'Holiday'
      },
      {
        id: 3,
        title: 'Old School',
        year: '2003',
        genre: 'Comedy'
      },
      {
        id: 4,
        title: 'Anchorman: The Legend of Ron Burgundy',
        year: '2004',
        genre: 'Comedy'
      },
      {
        id: 5,
        title: 'Stranger than Fiction',
        year: '2006',
        genre: 'Romantic Comedy'
      }
    ];

    this.setState({ movies: _.map(movies, this.mapMovieToDraggableMovieElement) });

  };

  render () {
    return <OrderableList items={this.state.movies} dropHandler={this.movieRankUpdate} containingTag="div"/>;
  }

}

export default MovieItemList;
```

We have hopefully achieved our goal of having a simple component to interface with. All we needed to do was supply our list elements, container tag, and drop handler.
Anytime one of our list elements is dropped into a new position our drop handler will be called with it's ID and new position.

To wrap up, we created a generic draggable item, a generic draggable list and used those 2 elements to create a custom draggable movie list.

I hope my first post was helpful and easy to understand. Please let me know if you have any suggestions or improvements. Glad to accept PR's against the example repository.
