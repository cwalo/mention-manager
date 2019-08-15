## Building the System

Okay, we've outlined the requirements and some of the considerations to keep in mind as we start building this thing. We could start building this on top of a controller we already have, but lets prototype this in a playground free from distractions. We know we'll need a text view on the screen, so let's start there.

```swift
final class MentionViewController: UIViewController {
  
  let textView = UITextView()
  
  // setup...
}
```

Now, we need an object to handle the delimiter searching, text replacement logic, etc. Your first thought might be to simply subclass `UITextView` and just do all of our work there, but I can foresee instances where we're already using a subclass (maybe from a third-party library) and can't reasonably extend it (who knows what the internals might be doing). We could potentially extend `UITextView` with this behavior, but there's probably some state to manage in this feature and extensions can't have stored properties (without having to deal with `objc_getAssociatedObject`), so lets not be clever. I'm thinking about an object that get's instantiated with a `UITextView` that provides callbacks to the owner of the textview. Let's see what this object might look like.

```swift

final class MentionManager {
  
  weak var textView: UITextView // weak reference to avoid a retain cycle
  
  init(textView: UITextView) {
    self.textView = textView
  }
}
```

And lets create an instance in our view controller. The lazy initializer is just for convenience.

```swift
final class MentionViewController: UIViewController {
  
  let textView = UITextView()
  
  lazy var mentionManager = MentionManager(textView: textView)
  
  // setup...
}
```

Before the MentionManager can do _anything_ cool, it needs to handle text events. The easiest thing would be to conform to `UITextViewDelegate` and add the respective protocol methods, but there's a problem with that. Another object might already be the delegate and the MentionManager would be hijacking those events. Lucky for us, `UITextView` posts notifications to `NotificationCenter` for _some_ events. The one we're most interested in is `textDidChangeNotification`. Let's add an observer for that one.

```swift

final class MentionManager {
  
  weak var textView: UITextView? // weak reference to avoid a retain cycle
  
  init(textView: UITextView) {
    self.textView = textView
    
    NotificationCenter.default.addObserver(self,
                                           selector: #selector(textViewTextDidChange),
                                           name: UITextView.textDidChangeNotification,
                                           object: textView)
  }
  
  @objc
  private func textViewTextDidChange(_ notification: Notification) {
    // text did change
  }
}
```

Okay, now what? As the text changes, we need to start looking for our delimiter. Easy enough.

```swift
  @objc
  private func textViewTextDidChange(_ notification: Notification) {
    guard let textView = textView else { return }
    guard let text = textView.text else { return }
    
    if text.last == "@" {
      // we found one!
      // capture the delimiter index
      // create a substring from the index to end of the text
    }
```

Not so fast! Hopefully you already see the pitfall in only checking the last character - there's no guarantee that user is only appending text. They could be inserting somewhere, so let's keep thinking. Here's a little sketch to help us.

I'll use "|" to represent the cursor.
```text
  This is some text. Oops, I want to insert|
  ...
  This is some text for @cor|. Oops, I want to insert
```

This is giving me some ideas. If we can get the index of the cursor, we can search backwards for the index of the closest delimiter... We can leverage a couple `UITextView` properties and methods to help us get the cursor index.
- `selectedTextRange: UITextRange`: "If the text range has a length, it indicates the currently selected text. If it has zero length, it indicates the caret (insertion point). If the text-range object is nil, it indicates that there is no current selection."
- `offset(from: UITextPosition, to toPosition: UITextPosition) -> Int`: "Return the number of UTF-16 characters between one text position and another text position."

```swift
final class MentionManager {
  ...
  
  private var cursorIndex: String.Index? {
    guard let textView = textView, let selectedRange = textView.selectedTextRange else {
        return nil
    }

    // "Return the number of UTF-16 characters between one text position and another text position."
    let cursorPosition = textView.offset(from: textView.beginningOfDocument, to: selectedRange.start)
    // create a String.Index using the cursorPosition to be used in creating a Range later
    return String.Index(utf16Offset: cursorPosition, in: textView.text)
  }
    
  ...

}
```

Great, back to our notification handler... Because Swift Strings are Collections, we can leverage Ranges and the `lastIndex(where:...)` function to find the closest delimiter preceding the `cursorIndex`.
```swift
  @objc
  private func textViewTextDidChange(_ notification: Notification) {
    guard let textView = textView else { return }
    guard let text = textView.text else { return }
    guard let cursorIndex = cursorIndex else { return }
    guard let lastDelimiterIndex = text[..<cursorIndex].lastIndex { $0 == "@" } else { return }
  }
```

At this point we can easily create a range representing the search term.
```swift
  @objc
  private func textViewTextDidChange(_ notification: Notification) {
    guard let textView = textView else { return }
    guard let text = textView.text else { return }
    guard let cursorIndex = cursorIndex else { return }
    guard let delimiterIndex = text[..<cursorIndex].lastIndex { $0 == "@" } else { return }
    
    let range = delimiterIndex..<cursorIndex
    let searchString = String(inputText[range])
  }
```
