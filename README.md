![Render Logo](Doc/logo.png)


[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Build](https://img.shields.io/badge/build-passing-green.svg?style=flat)](#)
[![Platform](https://img.shields.io/badge/platform-ios-lightgrey.svg?style=flat)](#)
[![Build](https://img.shields.io/badge/license-MIT-blue.svg?style=flat)](https://opensource.org/licenses/MIT)

*React-inspired swift library for writing UIKit UIs which are functions of their state.*

#Why

From [Why React matters](http://joshaber.github.io/2015/01/30/why-react-native-matters/):

[The framework] lets us write our UIs as a pure function of their state.


Right now we write UIs by poking at them, manually mutating their properties when something changes, adding and removing views, etc. This is fragile and error-prone. Some tools exist to lessen the pain, but they can only go so far. UIs are big, messy, mutable, stateful bags of sadness.

[The framework] let us describe our entire UI for a given state, and then it does the hard work of figuring out what needs to change. It abstracts all the fragile, error-prone code out away from us. 

## Installation

### Carthage



To install Carthage, run (using Homebrew):

```bash
$ brew update
$ brew install carthage
```

Then add the following line to your `Cartfile`:

```
github "alexdrone/Render" "master"    
```

#TL;DR

Render's building blocks are *Components* (described in the protocol `ComponentViewType`).
Despite virtually any `UIView` object can be a component (as long as it conforms to the above-cited protocol),
Render's core functionalities are exposed by the two main Component base classes: `ComponentView` and `StaticComponentView` (optimised for components that have a static view hierarchy).

Render layout engine is based on [FlexboxLayout](https://github.com/alexdrone/FlexboxLayout).

This is what a component (and its state) would look like:


```swift

struct Album: ComponentStateType {
	let title: String
	let artist: String
	let cover: UIImage  
	let featured: Bool
}

// COMPONENT
class AlbumComponentView: ComponentView {
    
    // the component state.
    var album: Album? {
        return self.state as? Album
    }
    
    // View as function of the state.
    override func construct() -> ComponentNodeType {
            
        return ComponentNode<UIView>().configure({
        		$0.style.flexDirection = self.album.featured ? .Row : .Column
            	$0.backgroundColor = UIColor.blackColor()

        }).children([
            
            ComponentNode<UIImageView>().configure({
				$0.image = self.album?.cover
				let size = self.album.featured ? self.parentSize.width : 48.0
				$0.style.dimensions = (size, size)
            }),
            
            ComponentNode<UIView>().configure({ 
            		$0.style.flexDirection = .Column
            		$0.style.margin = (8.0, 8.0, 8.0, 8.0, 0.0, 0.0)
                
            }).children([
                
                ComponentNode<UILabel>().configure({ 
                		$0.text = self.album?.title ?? "None"
                		$0.font = UIFont.systemFontOfSize(18.0, weight: UIFontWeightBold)
                		$0.textColor = UIColor.whiteColor()
                }),
                
                ComponentNode<UILabel>().configure({ view in
                		$0.text = self.album?.artist ?? "Uknown Artist"
                		$0.font = UIFont.systemFontOfSize(12.0, weight: UIFontWeightLight)
                		$0.textColor = UIColor.whiteColor()                		
                })
            ]),
         
            // This node will be part of the tree only when featured == false. *
            when(!self.album?.featured, ComponentNode<UILabel>().configure({ view in
                $0.style.justifyContent = .FlexEnd
                $0.text = "2016"
                $0.textColor = UIColor.whiteColor()
            }))

        ])
    }
    
}

```
<sup> * This can also be accomplished with a `StaticComponentView` by setting the `hidden` property in the configuration closure. </sup>


The view description is defined by the `construct()` method.
`ComponentNode<T>` is an abstaction around views of any sort that knows how to build, configure and layout the view when necessary.
Every time `renderComponent()` is called, a new tree is constructed, compared to the existing tree and only the required changes to the actual view hierarchy are performed - *if you have a static view hierarchy, you might want to inherit from `StaticComponentView` to skip this part of the rendering* . Also the `configure` closure passed as argument is re-applied to every view defined in the `construct()` method and the layout is re-computed based on the nodes' flexbox attributes. 

The component above would render to:

<p align="center">
<img src="Doc/render.jpg" width="900">

```swift

let albumComponent = AlbumComponentView()
albumComponentView.renderComponent()
```

#Lightweight

###Integration with UIKit

*Components* are plain UIViews, so they can be used inside a vanilla view hierarchy with *autolayout* or *layoutSubviews*.
Similiarly plain vanilla UIViews (UIKit components or custom ones) can be wrapped in a `ComponentNode` (so they can be part of a `ComponentView` or a `StaticComponentView`).

The framework doesn't force you to use the Component abstraction. You can use normal UIViews with autolayout inside a component or vice versa. This is probably one of the biggest difference from Facebook's `ComponentKit`.

###Thread model

Render's `renderComponent()` function is performed on the main thread. Diff+Layout+Configuration runs usually under 16ms, which makes it suitable for cells implementation (still keeping a smooth scrolling).


###Backend-driven UIs

Given the descriptive nature of Render's components, components can be defined in JSON or XML files and downloaded on-demand.
*The ComponentDeserializer is being worked on as we speak*.


#Cells 

You can wrap your components in `ComponentTableViewCell` or `ComponentCollectionViewCell` and use the classic dataSource/delegate pattern for you view controller.


```swift
class ViewControllerWithTableView: UIViewController, UITableViewDataSource, UITableViewDelegate {  
    var tableView: UITableView!
    var posts: [Post] = ... 
  
    override func viewDidLoad() {
        super.viewDidLoad()
  
        //ComponentTableViewCell works with 'UITableViewAutomaticDimension'
        self.tableView.rowHeight = UITableViewAutomaticDimension
        self.tableView.estimatedRowHeight = //..Setting this will dramatically improve reloadData() perf

        self.tableView.dataSource = self
        self.tableView.delegate = self
        self.view.addSubview(self.tableView)
    }

    func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return self.posts.count
    }
    
    func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
        self.posts[indexPath.row]... //change the state for the selected index path

        //render the component for the cell at the given index
        self.tableView.renderComponentAtIndexPath(indexPath)
    }
    
    func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
      let reuseIdentifier = "PostComponentCell"
      let cell: ComponentCell! =  
            //dequeue a cell with the given identifier 
            //(remember to use different identifiers for different component classes)
            tableView.dequeueReusableCellWithIdentifier(reuseIdentifier) as? ComponentCell ??
        
            //or create a new Cell wrapping the component
            ComponentCell(reuseIdentifier: reuseIdentifier, component: PostComponent())
                  
      //set the state for the cell
      cell.state = self.posts[indexPath.row]
      
      //and render the component
      cell.renderComponent(CGSize(tableView.bounds.size.width))
        
      return cell
    }
}
```


#Credits

- [React](https://github.com/facebook/react): The React github page
- [Few.swift](https://github.com/joshaber/Few.swift): Another React port for Swift. Check it out!
- [css-layout](https://github.com/facebook/css-layout): This project used the C src code for the flexbox layout engine.
- [Dwifft](https://github.com/jflinter/Dwifft): A swift array diff algorithm implmentation.
- [Backend-driven native UIs](https://www.facebook.com/atscaleevents/videos/1708052886134475/) from [JohnSundell](https://github.com/JohnSundell): A inspiring video about component-driven UIs (the demo project is also inspired from Spotify's UI).


