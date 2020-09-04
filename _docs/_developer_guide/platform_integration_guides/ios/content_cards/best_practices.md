---
nav_title: Best Practices and Use Cases
platform: iOS
page_order: 7
description: "This developer article covers Content Card best practices, three custom use case demo apps built out by our team, accompanying code snippets, as well as an implementation walkthrough."
---

# Best Practices and Use Cases
{% include video.html id="3h5Xbhl-TxE" align="right" %}

> This developer article covers Content Card best practices, three custom use case demo apps built out by our team, accompanying code snippets, as well as an implementation walkthrough. Included to the right is our full video walkthrough, with sections included and referenced throughout the rest of the article.

## Best Practices

### Design Considerations
<br>
When adding Content Cards to your codebase, create code that is decoupled. For example, try to minimize your SDK imports down to a single "import Appboy-iOS-SDK" statement. This limits issues that arise from excessive SDK imports, making it easier to track, debug, and alter code.
- __Decoupled__ - The "content" of the customer's data should be source-independent. Decoupled code allows you to easily track where your Content Card data is coming from and going to.
- __Flexibility__ - Offers type-agnostic extended functionality. You can extend your custom production code to handle Content Card data.
- __Easy to Debug__ - Errors can be traced to one place. Because of the minimization of SDK imports, your code now has a single source of data to check and debug along. 

## Demo Use Case Applications

Provided below are three instructional customer use case videos. We have provided code snippets for each demo application, as well as dashboard implementation examples.

### Supplemental Content

Content Cards can be implemented to seamlessly blend into an existing feed, allowing for data from multiple feeds to load simultaneously. This creates a cohesive, harmonious experience with Braze Content Cards and existing feed content.

{% tabs %}
{% tab Video %}

{% include video.html id="3h5Xbhl-TxE" align="center" %}

{% endtab %}
{% tab Swift - Demo Code Snippets %}
#### Load Content Cards Alongside Existing Content
Get the Data Simultaneously via Operation Cues
```swift
addOperation { [weak self] in
      guard let self = self else { return }
      self.loadCourses(self.courseCompletionHandler)
    }
    
addOperation { [weak self] in
      guard let self = self else { return }
      self.loadContentCards()
      self.semaphore.wait()
    }
```
Course Operation
```swift
func loadCourses(_ completion: @escaping ([Course]) -> ()) {
      switch result {
      case .success(let metaData):
        completion(metaData.courses)
      case .failure:
        cancelAllOperations()
    }
  }
```
Content Card Operation
```swift
func loadContentCards() {
    AppboyManager.shared.addObserverForContentCards(observer: self, selector: #selector(contentCardsUpdated))
    AppboyManager.shared.requestContentCardsRefresh()
  }

@objc private func contentCardsUpdated(_ notification: Notification) {
    let contentCards = AppboyManager.shared.handleContentCardsUpdated(notification, for: [.item(.course), .ad])
    contentCardCompletionHandler(contentCards)
    semaphore.signal()
  }
```

#### Extending Course Functionality
Populating Course Object via Content Card Payload
```swift
// MARK: - Course
struct Course: ContentCardable, Purchasable, Codable, Hashable {
  private(set) var contentCardData: ContentCardData?
  let id: Int
  let title: String
  let price: Decimal
  let imageUrl: String
    
  private enum CodingKeys: String, CodingKey {
    case id
    case title
    case price
    case imageUrl = "image"
  }
}
```

#### ABKContentCard Dependencies
The Only Dependencies on ABKContentCard are its Primitive Types
```swift
protocol ContentCardable {
  var contentCardData: ContentCardData? { get }
  init?(metaData: [ContentCardKey: Any], classType contentCardClassType: ContentCardClassType)
}


extension ContentCardable {
  var isContentCard: Bool {
    return contentCardData != nil
  }

// MARK: - ContentCardData
struct ContentCardData: Hashable {
  let contentCardId: String
  let contentCardClassType: ContentCardClassType
  let createdAt: Double
  let isDismissable: Bool
}
```

#### Campaign Key-Value Pairs from Braze Dasbboard
```swift
// MARK: - Content Card Initalizer
extension Course {
  init?(metaData: [ContentCardKey: Any], classType contentCardClassType: ContentCardClassType) {
    guard let idString = metaData[.idString] as? String,
      let createdAt = metaData[.created] as? Double,
      let isDismissable = metaData[.dismissable] as? Bool,
      let extras = metaData[.extras] as? [AnyHashable: Any],
      let title  = extras["course_title"] as? String,
      let detail = extras["course_detail"] as? String,
      let priceString = extras["course_price"] as? String,
      let price = Decimal(string: priceString),
      let imageUrl = extras["course_image"] as? String
      else { return nil }
```

#### Identifying Multiple Types
```swift
init(rawType: String?) {
    switch rawType?.lowercased() {
    case "ad":
      self = .ad
    case "coupon_code":
      self = .coupon
    case "course_recommendation":
      self = .item(.course)
    case "message_classic":
      self = .message(.fullPage)
    case "message_webview":
      self = .message(.webView)
    default:
      self = .none
    }
  }
```
{% endtab %}
{% endtabs %}

### Message Center
Content Cards can be used in a message center format where each message is its own Content Card. Each Content Card contains additional key-value pairs that power on-click UI/UX.
{% tabs %}
{% tab Video %}

{% include video.html id="3h5Xbhl-TxE" align="center" %}

{% endtab %}
{% tab Swift - Demo Code Snippets %}
#### Using Class_type for On Click Behavior
How to Filter and Identify Various Class Types
```swift
func addContentCardToView(with message: Message) {
    switch message.contentCardData?.contentCardClassType {
    case .message(.fullPage):
      loadContentCardFullPageView(with: message as! FullPageMessage)
    case .message(.webView):
      loadContentCardWebView(with: message as! WebViewMessage)
    default:
      break
    }
}
```
{% endtab %}
{% endtabs %}

### Interact-able View
Content Cards can be leveraged to create interactive experiences for your users. In the demo provided below, we have a Content Card pop-up appear at checkout providing users last-minute promotion opportunities. Well placed Cards like this are a great way to give users a "nudge" toward specific user actions. 

{% tabs %}
{% tab Video %}

{% include video.html id="3h5Xbhl-TxE" align="center" %}

{% endtab %}
{% tab Swift - Demo Code Snippets %}
#### Interactable View Steps

Requesting Content Cards
```swift
func loadContentCards() {
    AppboyManager.shared.addObserverForContentCards(observer: self, selector: #selector(contentCardsUpdated))
    AppboyManager.shared.requestContentCardsRefresh()
  }
```

Getting Type-Specific Content Cards
```swift
  @objc func contentCardsUpdated(_ notification: Notification) {
    guard let contentCards = AppboyManager.shared.handleContentCardsUpdated(notification, for: [.coupon]) as? [Coupon], !contentCards.isEmpty else { return }
}
```
#### Coupon Configuration
The Content Card Key-Value Pair provides the imageUrl for the View
```swift
func configureView(_ imageUrl: String?, _ origin: CGPoint, _ animatePoint: CGFloat, _ direction: UISwipeGestureRecognizer.Direction, _ delegate: GestureViewEventDelegate? = nil) {
    swipeGesture.direction = direction
    frame.origin = origin
    self.delegate = delegate
    
    configureView(imageUrl)
    
    UIView.animate(withDuration: 1.0, animations: {
      self.frame.origin.x = animatePoint
    })
  }
```
{% endtab %}
{% endtabs %}

## Implementation Walkthrough

Though we encourage minimal use of the ABKObjects, implementing Content Cards while still decoupling impressions, logging clicks and dismissals is a must. Below we have outlined a possible way to go about your implementation.

{% tabs %}
{% tab Video %}

{% include video.html id="3h5Xbhl-TxE" align="center" %}

{% endtab %}
{% tab Swift - Demo Code Snippets %}
#### __Implementation Components__


__Update Content Card Dictionary__<br>
Mapped via Content Card's idString
```swift
private var contentCardsDictionary: [String: ABKContentCard] = [:]

for card in cards {
      contentCardsDictionary[card.idString] = card
```


__Encapsulated ABK Methods__<br>
Only exposing the idString as a parameter
```swift
switch rows[indexPath.row] {
    case .item(let course):
      guard course.isContentCard else { break }
      course.logContentCardImpression()
```


__Get Content Card from IDString__<br>
Via Content Card Disctionary
```swift
protocol ContentCardable {
  func logContentCardImpression() {
    AppboyManager.shared.logContentCardImpression(idString: contentCardData?.contentCardId)
  }
}
```


__Call ABKCONTENTCARD Functions__<br>
100% handled via AppboyManager.Swift
```swift
func logContentCardClicked(idString: String?) {
    guard let idString = idString, let contentCard = contentCardsDictionary[idString] else { return }
    
    contentCard.logContentCardClicked()
  }
```
{% endtab %}
{% endtabs %}
