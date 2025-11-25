# V0Swift
SwiftUI Tips based on Vercel's React write up for iOS
[v0-insights-swiftui-translation-VERIFIED.md](https://github.com/user-attachments/files/23752674/v0-insights-swiftui-translation-VERIFIED.md)
# Translating v0 iOS App Insights to SwiftUI for iOS 26

## Summary

Vercel's engineering team built an iOS app "worthy of an Apple Design Award" using React Native. This writeup translates their hard-won insights, patterns, and solutions into native SwiftUI for iOS 26, leveraging the latest Liquid Glass design system and APIs from WWDC 2025. 

It was made using Claude Code + an Apple documentation MCP to verify code/advice accuracy. 

*happy to take this down if it's jacked up to post something like this*

---

## Table of Contents

1. [Architecture Philosophy](#1-architecture-philosophy)
2. [Building a Composable Chat](#2-building-a-composable-chat)
3. [Message Animation System](#3-message-animation-system)
4. [The "Blank Size" Problem](#4-the-blank-size-problem)
5. [Keyboard Management](#5-keyboard-management)
6. [Floating Composer with Liquid Glass](#6-floating-composer-with-liquid-glass)
7. [Streaming Content with Staggered Fade](#7-streaming-content-with-staggered-fade)
8. [Native Menus and Sheets](#8-native-menus-and-sheets)
9. [Markdown Rendering](#9-markdown-rendering)
10. [Performance Considerations](#10-performance-considerations)

---

## 1. Architecture Philosophy

### Vercel's Insight
> "We did not set out to build a mobile IDE with feature parity with our website. Instead, we wanted to build a simple, delightful experience for using AI to make things on the go."

### SwiftUI Translation

Build for the platform's strengths. iOS 26's Liquid Glass is designed for focus and delight—not information density. Your chat should feel like Apple Notes meets iMessage.

```swift
// The mental model: Your app is a thin, beautiful wrapper over your API
// Let the server do the heavy lifting; let SwiftUI create delight

@main
struct V0ChatApp: App {
    var body: some Scene {
        WindowGroup {
            ChatContainerView()
        }
    }
}
```

---

## 2. Building a Composable Chat

### Vercel's Approach (React Native)
They structured chat as composable plugins with multiple context providers:
- `ComposerHeightProvider`
- `MessageListProvider`  
- `NewMessageAnimationProvider`
- `KeyboardStateProvider`

### SwiftUI Translation: @Observable + Environment

iOS 26 with Swift's `@Observable` macro eliminates the need for complex provider hierarchies. SwiftUI automatically tracks property access and updates views efficiently.

```swift
import SwiftUI
import Observation

// MARK: - Observable State Container
@Observable
final class ChatState {
    var messages: [ChatMessage] = []
    var composerHeight: CGFloat = 0
    var keyboardHeight: CGFloat = 0
    var isMessageSendAnimating: Bool = false
    var blankSize: CGFloat = 0
    
    // Derived state
    var totalBottomInset: CGFloat {
        blankSize + composerHeight + keyboardHeight
    }
}

// MARK: - Message Model
struct ChatMessage: Identifiable, Equatable {
    let id: UUID
    let role: MessageRole
    var content: String
    var isStreaming: Bool = false
    let timestamp: Date
    
    enum MessageRole: Equatable {
        case user
        case assistant
        case optimisticPlaceholder
    }
}

// MARK: - Chat Container
struct ChatContainerView: View {
    @State private var chatState = ChatState()
    
    var body: some View {
        ChatMessagesView()
            .environment(chatState)
    }
}
```

### The Plugin Architecture in SwiftUI

Instead of hooks, use ViewModifiers that compose behavior:

```swift
// MARK: - Composable Modifiers (equivalent to RN hooks)
struct ChatMessagesView: View {
    @Environment(ChatState.self) private var state
    
    var body: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 12) {
                    ForEach(state.messages) { message in
                        MessageView(message: message)
                            .id(message.id)
                    }
                }
                .padding(.horizontal, 16)
            }
            // Composable behaviors as modifiers
            .keyboardAwareScrolling(proxy: proxy)
            .initialScrollToEnd(proxy: proxy)
            .animateNewMessages()
        }
        .safeAreaInset(edge: .bottom) {
            FloatingComposer()
        }
    }
}

// MARK: - Keyboard Aware Scrolling Modifier
struct KeyboardAwareScrollingModifier: ViewModifier {
    let proxy: ScrollViewProxy
    @Environment(ChatState.self) private var state
    
    func body(content: Content) -> some View {
        content
            .onReceive(NotificationCenter.default.publisher(for: UIResponder.keyboardWillShowNotification)) { notification in
                handleKeyboardShow(notification)
            }
            .onReceive(NotificationCenter.default.publisher(for: UIResponder.keyboardWillHideNotification)) { _ in
                handleKeyboardHide()
            }
    }
    
    private func handleKeyboardShow(_ notification: Notification) {
        guard let keyboardFrame = notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else { return }
        
        withAnimation(.interpolatingSpring(stiffness: 500, damping: 45)) {
            state.keyboardHeight = keyboardFrame.height
        }
    }
    
    private func handleKeyboardHide() {
        withAnimation(.interpolatingSpring(stiffness: 500, damping: 45)) {
            state.keyboardHeight = 0
        }
    }
}

extension View {
    func keyboardAwareScrolling(proxy: ScrollViewProxy) -> some View {
        modifier(KeyboardAwareScrollingModifier(proxy: proxy))
    }
}
```

---

## 3. Message Animation System

### Vercel's Insight
> "When you send a message on v0, the message bubble smoothly fades in and slides to the top. Immediately after the user message is done animating, the assistant messages fade in."

They used:
- Reanimated shared values for state without re-renders
- `useFirstMessageAnimation` hook for measuring and animating
- Synchronized completion callbacks between animations

### SwiftUI Translation: PhaseAnimator + Namespace

iOS 17+ introduced `PhaseAnimator` for exactly this kind of sequenced animation. iOS 26 enhances it with the `@Animatable` macro.

```swift
// MARK: - Animation Phases
enum MessageAnimationPhase: CaseIterable {
    case hidden
    case appearing
    case visible
    
    var opacity: Double {
        switch self {
        case .hidden: 0
        case .appearing: 0.5
        case .visible: 1
        }
    }
    
    var offset: CGFloat {
        switch self {
        case .hidden: 50
        case .appearing: 10
        case .visible: 0
        }
    }
    
    var scale: CGFloat {
        switch self {
        case .hidden: 0.95
        case .appearing: 0.98
        case .visible: 1
        }
    }
}

// MARK: - User Message with First Message Animation
struct UserMessageView: View {
    let message: ChatMessage
    let isFirstMessage: Bool
    @Environment(ChatState.self) private var state
    
    @State private var animationComplete = false
    
    var body: some View {
        HStack {
            Spacer()
            
            Text(message.content)
                .padding(.horizontal, 16)
                .padding(.vertical, 12)
                .background(.blue, in: .rect(cornerRadius: 20))
                .foregroundStyle(.white)
        }
        .modifier(FirstMessageAnimationModifier(
            isEnabled: isFirstMessage && state.isMessageSendAnimating,
            onComplete: {
                animationComplete = true
                state.isMessageSendAnimating = false
            }
        ))
    }
}

// MARK: - First Message Animation Modifier
struct FirstMessageAnimationModifier: ViewModifier {
    let isEnabled: Bool
    let onComplete: () -> Void
    
    @State private var phase: MessageAnimationPhase = .hidden
    
    func body(content: Content) -> some View {
        content
            .opacity(isEnabled ? phase.opacity : 1)
            .offset(y: isEnabled ? phase.offset : 0)
            .scaleEffect(isEnabled ? phase.scale : 1)
            .onAppear {
                guard isEnabled else { return }
                
                // Sequence the animations
                withAnimation(.spring(duration: 0.3, bounce: 0.2)) {
                    phase = .appearing
                }
                
                withAnimation(.spring(duration: 0.4, bounce: 0.15).delay(0.2)) {
                    phase = .visible
                } completion: {
                    onComplete()
                }
            }
    }
}

// MARK: - Assistant Message with Fade After User
struct AssistantMessageView: View {
    let message: ChatMessage
    let isFirstAssistantMessage: Bool
    @Environment(ChatState.self) private var state
    
    @State private var isVisible = false
    
    var body: some View {
        HStack {
            StreamingTextView(text: message.content, isStreaming: message.isStreaming)
                .padding(.horizontal, 16)
                .padding(.vertical, 12)
                .background(.secondary.opacity(0.1), in: .rect(cornerRadius: 20))
            
            Spacer()
        }
        .opacity(shouldAnimate ? (isVisible ? 1 : 0) : 1)
        .onChange(of: state.isMessageSendAnimating) { _, newValue in
            // Fade in after user message animation completes
            if !newValue && isFirstAssistantMessage {
                withAnimation(.easeOut(duration: 0.35)) {
                    isVisible = true
                }
            }
        }
        .onAppear {
            // For existing chats, show immediately
            if !shouldAnimate {
                isVisible = true
            }
        }
    }
    
    private var shouldAnimate: Bool {
        isFirstAssistantMessage && !isVisible
    }
}
```

### Swift Enhancement: @Animatable Macro

```swift
// The @Animatable macro (available iOS 13.0+) provides cleaner animation code
// It synthesizes Animatable conformance and animatableData automatically
@Animatable
struct MessageAppearanceEffect: ViewModifier {
    var progress: Double // 0 = hidden, 1 = visible
    
    // @Animatable synthesizes animatableData using the animatable properties
    
    func body(content: Content) -> some View {
        content
            .opacity(progress)
            .offset(y: (1 - progress) * 30)
            .scaleEffect(0.95 + (0.05 * progress))
    }
}
```

---

## 4. The "Blank Size" Problem

### Vercel's Insight
This was one of their biggest challenges:
> "When you send a message, the message bubble smoothly fades in and slides **to the top**... We needed a strategy to push the user message to the top of the chat. We referred to this as 'blank size'."

They tried:
- View at bottom of ScrollView ❌
- Bottom padding on ScrollView ❌
- TranslateY on content ❌
- Minimum height on last message ❌

**Solution:** `contentInset` on native `UIScrollView` paired with `scrollToEnd({ offset })`

### SwiftUI Translation: safeAreaPadding + contentMargins

iOS 17+ provides native solutions that map directly to UIKit's `contentInset`:

```swift
// MARK: - Blank Size Calculator
struct BlankSizeCalculator {
    let containerHeight: CGFloat
    let keyboardHeight: CGFloat
    let lastUserMessageHeight: CGFloat
    let lastAssistantMessageHeight: CGFloat
    let composerHeight: CGFloat
    
    var blankSize: CGFloat {
        let visibleHeight = containerHeight - keyboardHeight - composerHeight
        let contentHeight = lastUserMessageHeight + lastAssistantMessageHeight
        
        // If content is shorter than visible area, add blank space
        // This pushes content to the top
        return max(0, visibleHeight - contentHeight)
    }
}

// MARK: - Chat View with Dynamic Blank Size
struct ChatMessagesListView: View {
    @Environment(ChatState.self) private var state
    @State private var containerHeight: CGFloat = 0
    
    var body: some View {
        GeometryReader { geometry in
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 12) {
                        ForEach(Array(state.messages.enumerated()), id: \.element.id) { index, message in
                            MessageView(message: message, index: index)
                                .id(message.id)
                                .background(
                                    GeometryReader { messageGeo in
                                        Color.clear.preference(
                                            key: MessageHeightPreference.self,
                                            value: [message.id: messageGeo.size.height]
                                        )
                                    }
                                )
                        }
                    }
                    .padding(.horizontal, 16)
                }
                // iOS 17+: Native content inset support
                .contentMargins(.bottom, state.blankSize, for: .scrollContent)
                // Additional safe area for composer
                .safeAreaPadding(.bottom, state.composerHeight)
            }
            .onAppear {
                containerHeight = geometry.size.height
            }
        }
    }
}

// MARK: - Preference Key for Height Tracking
struct MessageHeightPreference: PreferenceKey {
    static var defaultValue: [UUID: CGFloat] = [:]
    
    static func reduce(value: inout [UUID: CGFloat], nextValue: () -> [UUID: CGFloat]) {
        value.merge(nextValue()) { $1 }
    }
}
```

### Advanced: Scroll Position Tracking (iOS 17+)

```swift
// MARK: - Modern Scroll Position API
struct ModernChatScrollView: View {
    @Environment(ChatState.self) private var state
    @State private var scrollPosition: ScrollPosition = .init()
    
    var body: some View {
        ScrollView {
            LazyVStack(spacing: 12) {
                ForEach(state.messages) { message in
                    MessageView(message: message, index: 0)
                        .id(message.id)
                }
            }
            .scrollTargetLayout()
        }
        .scrollPosition($scrollPosition)
        .contentMargins(.bottom, state.blankSize, for: .scrollContent)
        .onChange(of: state.messages.count) { _, _ in
            // Scroll to end when new message arrives
            if let lastMessage = state.messages.last {
                withAnimation(.spring(duration: 0.3)) {
                    scrollPosition.scrollTo(id: lastMessage.id, anchor: .top)
                }
            }
        }
    }
}
```

---

## 5. Keyboard Management

### Vercel's Insight
> "Building a good chat experience hinges on elegant keyboard handling. Achieving native feel in this area was tedious and challenging... every time a new iOS beta version came out, our chat seemingly broke entirely."

Their `useKeyboardAwareMessageList` was ~1,000 lines handling:
- Shrinking blank size when keyboard opens
- Shifting content up contextually
- Interactive keyboard dismissal
- Edge cases with app backgrounding

### SwiftUI Translation: Native + Manual Hybrid

iOS 26 has improved keyboard handling, but for complete control you'll still need some manual work:

```swift
import SwiftUI
import Combine

// MARK: - Keyboard Observer
@Observable
final class KeyboardObserver {
    var height: CGFloat = 0
    var isVisible: Bool = false
    var animationDuration: TimeInterval = 0.25
    var animationCurve: UIView.AnimationCurve = .easeInOut
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        setupObservers()
    }
    
    private func setupObservers() {
        NotificationCenter.default.publisher(for: UIResponder.keyboardWillShowNotification)
            .sink { [weak self] notification in
                self?.handleKeyboardWillShow(notification)
            }
            .store(in: &cancellables)
        
        NotificationCenter.default.publisher(for: UIResponder.keyboardWillHideNotification)
            .sink { [weak self] notification in
                self?.handleKeyboardWillHide(notification)
            }
            .store(in: &cancellables)
        
        // iOS 26: Handle the triple-fire bug when app returns from background
        NotificationCenter.default.publisher(for: UIApplication.didBecomeActiveNotification)
            .debounce(for: .milliseconds(100), scheduler: RunLoop.main)
            .sink { [weak self] _ in
                self?.validateKeyboardState()
            }
            .store(in: &cancellables)
    }
    
    private func handleKeyboardWillShow(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let keyboardFrame = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval,
              let curveValue = userInfo[UIResponder.keyboardAnimationCurveUserInfoKey] as? Int,
              let curve = UIView.AnimationCurve(rawValue: curveValue) else {
            return
        }
        
        animationDuration = duration
        animationCurve = curve
        height = keyboardFrame.height
        isVisible = true
    }
    
    private func handleKeyboardWillHide(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval else {
            return
        }
        
        animationDuration = duration
        height = 0
        isVisible = false
    }
    
    private func validateKeyboardState() {
        // Verify keyboard state matches what we think it is
        // This handles the iOS bug where keyboard events fire multiple times
    }
}

// MARK: - Keyboard Aware View Modifier
struct KeyboardAwareModifier: ViewModifier {
    @Environment(KeyboardObserver.self) private var keyboard
    @Environment(ChatState.self) private var state
    
    let scrollProxy: ScrollViewProxy
    let isScrolledToEnd: Bool
    
    func body(content: Content) -> some View {
        content
            .onChange(of: keyboard.height) { oldHeight, newHeight in
                handleKeyboardChange(from: oldHeight, to: newHeight)
            }
    }
    
    private func handleKeyboardChange(from oldHeight: CGFloat, to newHeight: CGFloat) {
        let isOpening = newHeight > oldHeight
        
        // Vercel's key insight: behavior depends on scroll position and blank size
        if isOpening {
            handleKeyboardOpening(newHeight: newHeight)
        } else {
            handleKeyboardClosing()
        }
    }
    
    private func handleKeyboardOpening(newHeight: CGFloat) {
        // Shrink blank size when keyboard opens
        let newBlankSize = max(0, state.blankSize - newHeight)
        
        withAnimation(.interpolatingSpring(stiffness: 500, damping: 45)) {
            state.blankSize = newBlankSize
            state.keyboardHeight = newHeight
        }
        
        // If scrolled to end, keep content pinned above keyboard
        if isScrolledToEnd, let lastMessage = state.messages.last {
            withAnimation(.interpolatingSpring(stiffness: 500, damping: 45)) {
                scrollProxy.scrollTo(lastMessage.id, anchor: .bottom)
            }
        }
    }
    
    private func handleKeyboardClosing() {
        withAnimation(.interpolatingSpring(stiffness: 500, damping: 45)) {
            state.keyboardHeight = 0
            // Recalculate blank size based on content
            recalculateBlankSize()
        }
    }
    
    private func recalculateBlankSize() {
        // Implementation based on current visible content
    }
}

// MARK: - Interactive Keyboard Dismissal
struct InteractiveKeyboardDismissalModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .scrollDismissesKeyboard(.interactively)
    }
}
```

### Native iOS 26 Approach (Simplified)

For many cases, iOS 26's native handling is sufficient:

```swift
struct SimplifiedChatView: View {
    @State private var messages: [ChatMessage] = []
    @FocusState private var isComposerFocused: Bool
    
    var body: some View {
        NavigationStack {
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack {
                        ForEach(messages) { message in
                            MessageView(message: message, index: 0)
                                .id(message.id)
                        }
                    }
                }
                // iOS 26: Much improved native keyboard handling
                .scrollDismissesKeyboard(.interactively)
                .defaultScrollAnchor(.bottom) // iOS 18+: Start scrolled to bottom
                .safeAreaInset(edge: .bottom) {
                    ComposerView()
                        .focused($isComposerFocused)
                }
            }
            .toolbar {
                // iOS 26: Use ToolbarSpacer to create logical groupings
                ToolbarItemGroup(placement: .topBarTrailing) {
                    Button("Edit", systemImage: "pencil") { }
                    Button("Share", systemImage: "square.and.arrow.up") { }
                }
                ToolbarSpacer(.fixed, placement: .topBarTrailing)
                ToolbarItem(placement: .topBarTrailing) {
                    Button("Done", systemImage: "checkmark") { }
                        .buttonStyle(.glassProminent)
                }
            }
        }
    }
}
```

---

## 6. Floating Composer with Liquid Glass

### Vercel's Insight
> "Inspired by iMessage's bottom toolbar in iOS 26, we built a Liquid Glass composer with a progressive blur."

They used `@callstack/liquid-glass` and `LiquidGlassContainerView` with interactive morphing.

### SwiftUI Translation: Native iOS 26 Liquid Glass

This is where SwiftUI shines—Apple's native implementation is even better than third-party solutions:

> ⚠️ **Important WWDC Guidance**: Avoid "glass-on-glass" — don't apply glass effects to content inside glass containers (sheets at small detents, toolbars). The system will automatically remove custom backgrounds behind bars to enable the scroll edge effect.

```swift
// MARK: - Glass Effect Variants
// .regular - Default glass with blur and reflection
// .regular.interactive() - Adds pressed state for tappable elements
// .regular.tint(.blue) - Adds color overlay
// .clear - Removes tint, increases transparency
// .identity - Removes glass effect entirely

// MARK: - Floating Composer with Liquid Glass
struct FloatingComposer: View {
    @Environment(ChatState.self) private var state
    @FocusState private var isFocused: Bool
    
    @State private var messageText = ""
    @State private var composerHeight: CGFloat = 0
    @Namespace private var composerNamespace
    
    var body: some View {
        GlassEffectContainer {
            HStack(spacing: 12) {
                // Attachment button
                Button {
                    // Handle attachments
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.title2)
                }
                .buttonStyle(.glass)
                .glassEffectID("attachButton", in: composerNamespace)
                
                // Text input
                TextField("Message", text: $messageText, axis: .vertical)
                    .lineLimit(1...6)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 10)
                    .glassEffect(.regular.interactive)
                    .glassEffectID("textField", in: composerNamespace)
                    .focused($isFocused)
                
                // Send button
                Button {
                    sendMessage()
                } label: {
                    Image(systemName: "arrow.up.circle.fill")
                        .font(.title2)
                }
                .buttonStyle(.glassProminent)
                .tint(.blue)
                .disabled(messageText.isEmpty)
                .glassEffectID("sendButton", in: composerNamespace)
            }
            .padding(.horizontal, 16)
            .padding(.vertical, 12)
        }
        .background(
            GeometryReader { geo in
                Color.clear.preference(
                    key: ComposerHeightPreference.self,
                    value: geo.size.height
                )
            }
        )
        .onPreferenceChange(ComposerHeightPreference.self) { height in
            state.composerHeight = height
        }
    }
    
    private func sendMessage() {
        guard !messageText.isEmpty else { return }
        
        let newMessage = ChatMessage(
            id: UUID(),
            role: .user,
            content: messageText,
            timestamp: Date()
        )
        
        state.isMessageSendAnimating = true
        state.messages.append(newMessage)
        messageText = ""
        
        // Add optimistic placeholder for assistant response
        let placeholder = ChatMessage(
            id: UUID(),
            role: .optimisticPlaceholder,
            content: "",
            timestamp: Date()
        )
        state.messages.append(placeholder)
    }
}

struct ComposerHeightPreference: PreferenceKey {
    static var defaultValue: CGFloat = 0
    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = nextValue()
    }
}
```

### Gesture-Based Focus (Swipe Up to Open Keyboard)

Vercel patched React Native to add this. In SwiftUI:

```swift
// MARK: - Swipe to Focus Modifier
struct SwipeToFocusModifier: ViewModifier {
    @FocusState.Binding var isFocused: Bool
    
    func body(content: Content) -> some View {
        content
            .gesture(
                DragGesture(minimumDistance: 20)
                    .onEnded { value in
                        // Swipe up to focus
                        if value.translation.height < -50 && 
                           value.velocity.height < -250 &&
                           !isFocused {
                            isFocused = true
                        }
                    }
            )
    }
}

extension View {
    func swipeToFocus(_ isFocused: FocusState<Bool>.Binding) -> some View {
        modifier(SwipeToFocusModifier(isFocused: isFocused))
    }
}
```

### Scrolling Content When Composer Height Changes

```swift
// MARK: - Composer Height Change Handler
struct ComposerHeightScrollModifier: ViewModifier {
    @Environment(ChatState.self) private var state
    let proxy: ScrollViewProxy
    
    @State private var previousHeight: CGFloat = 0
    @State private var isNearBottom = true
    
    func body(content: Content) -> some View {
        content
            .onChange(of: state.composerHeight) { oldHeight, newHeight in
                // Only scroll if near bottom (like Vercel's implementation)
                guard isNearBottom, newHeight > oldHeight else { return }
                
                if let lastMessage = state.messages.last {
                    withAnimation(.easeOut(duration: 0.1)) {
                        proxy.scrollTo(lastMessage.id, anchor: .bottom)
                    }
                }
            }
    }
}
```

---

## 7. Streaming Content with Staggered Fade

### Vercel's Insight
> "When an AI's assistant message streams in, it needs to feel smooth."

They created:
- `<FadeInStaggeredIfStreaming />` for elements
- `<TextFadeInStaggeredIfStreaming />` for text
- A pool system limiting concurrent animations
- Word chunking with 32ms delays between batches

### SwiftUI Translation: Streaming Text View with Staggered Animation

```swift
// MARK: - Streaming Text View
struct StreamingTextView: View {
    let text: String
    let isStreaming: Bool
    
    @State private var displayedText: String = ""
    @State private var wordQueue: [String] = []
    @State private var animatingWords: Set<Int> = []
    
    // Pool limit - max 4 words animating at once (like Vercel)
    private let maxConcurrentAnimations = 4
    private let staggerDelay: TimeInterval = 0.032 // 32ms like Vercel
    
    var body: some View {
        Text(attributedDisplayedText)
            .onChange(of: text) { oldText, newText in
                handleTextChange(from: oldText, to: newText)
            }
            .onAppear {
                if !isStreaming {
                    // Show immediately for existing messages
                    displayedText = text
                }
            }
    }
    
    private var attributedDisplayedText: AttributedString {
        var attributed = AttributedString(displayedText)
        // Add any markdown styling here
        return attributed
    }
    
    private func handleTextChange(from oldText: String, to newText: String) {
        guard isStreaming else {
            displayedText = newText
            return
        }
        
        // Find new words
        let oldWords = oldText.split(separator: " ").map(String.init)
        let newWords = newText.split(separator: " ").map(String.init)
        
        let addedWords = Array(newWords.dropFirst(oldWords.count))
        
        // Add to queue and process
        wordQueue.append(contentsOf: addedWords)
        processWordQueue()
    }
    
    private func processWordQueue() {
        guard animatingWords.count < maxConcurrentAnimations,
              !wordQueue.isEmpty else { return }
        
        let wordsToAnimate = min(2, wordQueue.count) // Batch of 2 like Vercel
        
        for i in 0..<wordsToAnimate {
            guard !wordQueue.isEmpty else { break }
            
            let word = wordQueue.removeFirst()
            let wordIndex = displayedText.split(separator: " ").count
            
            animatingWords.insert(wordIndex)
            
            // Staggered appearance
            DispatchQueue.main.asyncAfter(deadline: .now() + Double(i) * staggerDelay) {
                withAnimation(.easeOut(duration: 0.5)) {
                    displayedText += (displayedText.isEmpty ? "" : " ") + word
                }
                
                // Remove from animating set after animation
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
                    animatingWords.remove(wordIndex)
                    processWordQueue() // Process next batch
                }
            }
        }
    }
}

// MARK: - Alternative: Per-Character Streaming (More Fluid)
struct CharacterStreamingTextView: View {
    let text: String
    let isStreaming: Bool
    
    @State private var visibleCharacterCount: Int = 0
    
    var body: some View {
        Text(visibleText)
            .onChange(of: text) { _, newText in
                animateNewCharacters(in: newText)
            }
            .onAppear {
                if isStreaming {
                    visibleCharacterCount = 0
                } else {
                    visibleCharacterCount = text.count
                }
            }
    }
    
    private var visibleText: String {
        String(text.prefix(visibleCharacterCount))
    }
    
    private func animateNewCharacters(in newText: String) {
        guard isStreaming else {
            visibleCharacterCount = newText.count
            return
        }
        
        // Animate in batches of 3-5 characters for smooth streaming
        let batchSize = 3
        let targetCount = newText.count
        
        Timer.scheduledTimer(withTimeInterval: 0.016, repeats: true) { timer in
            if visibleCharacterCount >= targetCount {
                timer.invalidate()
                return
            }
            
            visibleCharacterCount = min(visibleCharacterCount + batchSize, targetCount)
        }
    }
}
```

### iOS 26 Enhancement: TextRenderer for Custom Effects

```swift
// MARK: - Custom Text Renderer for Streaming Effect
struct StreamingTextRenderer: TextRenderer {
    var progress: Double // 0...1 representing visible portion
    
    func draw(layout: Text.Layout, in context: inout GraphicsContext) {
        let totalCharacters = layout.flatMap { $0 }.count
        let visibleCount = Int(Double(totalCharacters) * progress)
        
        var characterIndex = 0
        
        for line in layout {
            for run in line {
                for glyph in run {
                    let isVisible = characterIndex < visibleCount
                    let fadeProgress = characterIndex < visibleCount - 3 ? 1.0 :
                                       Double(visibleCount - characterIndex) / 3.0
                    
                    context.opacity = isVisible ? fadeProgress : 0
                    context.draw(glyph)
                    characterIndex += 1
                }
            }
        }
    }
}
```

---

## 8. Native Menus and Sheets

### Vercel's Insight
> "For menus, we used Zeego, which relies on react-native-ios-context-menu to render the native UIMenu under the hood. Zeego automatically renders Liquid Glass menus when you build with Xcode 26."

### SwiftUI Translation: Built-in Liquid Glass

SwiftUI in iOS 26 gets Liquid Glass menus automatically:

```swift
// MARK: - Context Menu with Liquid Glass (Automatic in iOS 26)
struct MessageContextMenuView: View {
    let message: ChatMessage
    
    var body: some View {
        MessageBubble(message: message)
            .contextMenu {
                Button {
                    UIPasteboard.general.string = message.content
                } label: {
                    Label("Copy", systemImage: "doc.on.doc")
                }
                
                Button {
                    // Share action
                } label: {
                    Label("Share", systemImage: "square.and.arrow.up")
                }
                
                Divider()
                
                Button(role: .destructive) {
                    // Delete action
                } label: {
                    Label("Delete", systemImage: "trash")
                }
            }
    }
}

// MARK: - Menu Button with Liquid Glass
struct OptionsMenuButton: View {
    @Namespace private var menuNamespace
    
    var body: some View {
        Menu {
            Button("New Chat", systemImage: "plus") { }
            Button("Settings", systemImage: "gear") { }
            Divider()
            Button("Help", systemImage: "questionmark.circle") { }
        } label: {
            Image(systemName: "ellipsis.circle.fill")
                .font(.title2)
        }
        .menuStyle(.button)
        .buttonStyle(.glass)
        .glassEffectID("optionsMenu", in: menuNamespace)
    }
}
```

### Sheets with iOS 26 Morphing

```swift
// MARK: - Sheet Presentation with Morphing Transition
// Note: For sheets to morph from source, use matchedTransitionSource
// on the presenting element. This works with iOS 18+ navigationTransition.
struct ChatSettingsSheet: View {
    @Binding var isPresented: Bool
    @Namespace private var sheetNamespace
    
    var body: some View {
        Button {
            isPresented = true
        } label: {
            Image(systemName: "gear")
        }
        .buttonStyle(.glass)
        .matchedTransitionSource(id: "settingsButton", in: sheetNamespace)
        .sheet(isPresented: $isPresented) {
            SettingsContentView()
                .presentationDetents([.medium, .large])
                // iOS 18+: Use navigationTransition for morph effect
                .navigationTransition(.zoom(sourceID: "settingsButton", in: sheetNamespace))
        }
    }
}

// MARK: - Bottom Sheet Alternative
struct ModernBottomSheet<Content: View>: View {
    @Binding var isPresented: Bool
    @ViewBuilder let content: () -> Content
    
    var body: some View {
        content()
            .sheet(isPresented: $isPresented) {
                sheetContent
                    .presentationDetents([.height(200), .medium, .large])
                    .presentationDragIndicator(.visible)
                    .presentationBackgroundInteraction(.enabled(upThrough: .medium))
                    // iOS 26: Liquid Glass sheet background
                    .presentationBackground(.thinMaterial)
            }
    }
    
    @ViewBuilder
    private var sheetContent: some View {
        VStack {
            // Sheet content
        }
    }
}
```

---

## 9. Markdown Rendering

### Vercel's Insight
> "Markdown is fast and supports dynamic components"

They used MDX components with `<FadeInStaggeredIfStreaming />` wrappers.

### SwiftUI Translation: AttributedString + iOS 26 Rich Text

iOS 26 enhances `TextEditor` with full `AttributedString` support:

```swift
// MARK: - Markdown Parser
struct MarkdownRenderer {
    static func render(_ markdown: String) -> AttributedString {
        do {
            var attributed = try AttributedString(
                markdown: markdown,
                options: .init(
                    allowsExtendedAttributes: true,
                    interpretedSyntax: .inlineOnlyPreservingWhitespace,
                    failurePolicy: .returnPartiallyParsedIfPossible
                )
            )
            
            // Apply custom styling
            attributed = applyCodeBlockStyling(to: attributed)
            
            return attributed
        } catch {
            return AttributedString(markdown)
        }
    }
    
    private static func applyCodeBlockStyling(to string: AttributedString) -> AttributedString {
        var result = string
        
        // Find and style inline code
        for run in result.runs {
            if run.inlinePresentationIntent?.contains(.code) == true {
                let range = run.range
                result[range].backgroundColor = .secondary.opacity(0.1)
                result[range].font = .system(.body, design: .monospaced)
            }
        }
        
        return result
    }
}

// MARK: - Streaming Markdown View
struct StreamingMarkdownView: View {
    let content: String
    let isStreaming: Bool
    
    @State private var renderedContent: AttributedString = AttributedString()
    
    var body: some View {
        Text(renderedContent)
            .textSelection(.enabled)
            .onChange(of: content, initial: true) { _, newContent in
                Task {
                    renderedContent = MarkdownRenderer.render(newContent)
                }
            }
    }
}

// MARK: - Code Block View with Syntax Highlighting
struct CodeBlockView: View {
    let code: String
    let language: String?
    
    @State private var isCopied = false
    
    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            // Header
            HStack {
                Text(language ?? "code")
                    .font(.caption)
                    .foregroundStyle(.secondary)
                
                Spacer()
                
                Button {
                    UIPasteboard.general.string = code
                    withAnimation { isCopied = true }
                    
                    DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                        withAnimation { isCopied = false }
                    }
                } label: {
                    Label(
                        isCopied ? "Copied" : "Copy",
                        systemImage: isCopied ? "checkmark" : "doc.on.doc"
                    )
                    .font(.caption)
                    .contentTransition(.symbolEffect(.replace))
                }
                .buttonStyle(.glass)
            }
            .padding(.horizontal, 12)
            .padding(.vertical, 8)
            
            // Code content
            ScrollView(.horizontal, showsIndicators: false) {
                Text(code)
                    .font(.system(.body, design: .monospaced))
                    .textSelection(.enabled)
                    .padding(12)
            }
        }
        .background(.quaternary.opacity(0.5), in: .rect(cornerRadius: 12))
    }
}
```

---

## 10. Performance Considerations

### Vercel's Insights
- Relied on `ref.current.measure()` being synchronous in React Native's New Architecture
- Used shared values to avoid re-renders during animations
- Limited concurrent animations via "pools"
- Multiple `scrollToEnd` calls needed due to dynamic heights

### SwiftUI Translation: Performance Best Practices

```swift
// MARK: - Performance Tips for Chat Apps

// 1. Use LazyVStack for message lists
struct PerformantMessageList: View {
    let messages: [ChatMessage]
    
    var body: some View {
        ScrollView {
            LazyVStack(spacing: 12) {  // Lazy = only renders visible items
                ForEach(messages) { message in
                    MessageView(message: message, index: 0)
                }
            }
        }
    }
}

// 2. Equatable conformance to prevent unnecessary redraws
struct MessageView: View, Equatable {
    let message: ChatMessage
    let index: Int
    
    static func == (lhs: MessageView, rhs: MessageView) -> Bool {
        lhs.message == rhs.message
    }
    
    var body: some View {
        // Message content
        Text(message.content)
    }
}

// 3. Animation Pool to Limit Concurrent Animations
actor AnimationPool {
    private var activeAnimations = 0
    private let maxConcurrent = 4
    private var waitingQueue: [CheckedContinuation<Void, Never>] = []
    
    func acquire() async {
        if activeAnimations < maxConcurrent {
            activeAnimations += 1
            return
        }
        
        await withCheckedContinuation { continuation in
            waitingQueue.append(continuation)
        }
        activeAnimations += 1
    }
    
    func release() {
        activeAnimations -= 1
        
        if let next = waitingQueue.first {
            waitingQueue.removeFirst()
            next.resume()
        }
    }
}

// 4. Debounced scroll operations
struct DebouncedScrollModifier: ViewModifier {
    let proxy: ScrollViewProxy
    @State private var scrollTask: Task<Void, Never>?
    
    func body(content: Content) -> some View {
        content
            .onChange(of: someValue) { _, _ in
                scrollTask?.cancel()
                scrollTask = Task {
                    try? await Task.sleep(nanoseconds: 16_000_000) // 16ms
                    guard !Task.isCancelled else { return }
                    
                    await MainActor.run {
                        withAnimation {
                            proxy.scrollTo(targetID, anchor: .bottom)
                        }
                    }
                }
            }
    }
    
    // Placeholder properties
    private var someValue: Int { 0 }
    private var targetID: String { "" }
}

// 5. Preference aggregation for height tracking
struct BatchedHeightPreference: PreferenceKey {
    static var defaultValue: [UUID: CGFloat] = [:]
    
    static func reduce(value: inout [UUID: CGFloat], nextValue: () -> [UUID: CGFloat]) {
        // Batch updates to minimize preference change callbacks
        value.merge(nextValue()) { _, new in new }
    }
}
```

---

## Complete Example: Putting It All Together

```swift
import SwiftUI
import Observation

// MARK: - Main Chat View
struct V0StyleChatView: View {
    @State private var chatState = ChatState()
    @State private var keyboardObserver = KeyboardObserver()
    @FocusState private var isComposerFocused: Bool
    
    var body: some View {
        GeometryReader { geometry in
            ZStack {
                // Background
                Color(.systemBackground)
                    .ignoresSafeArea()
                
                // Chat content
                VStack(spacing: 0) {
                    // Messages
                    chatMessages
                    
                    // Composer (positioned via safeAreaInset)
                }
            }
        }
        .environment(chatState)
        .environment(keyboardObserver)
    }
    
    private var chatMessages: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 16) {
                    ForEach(Array(chatState.messages.enumerated()), id: \.element.id) { index, message in
                        MessageView(message: message, index: index)
                            .id(message.id)
                    }
                }
                .padding(.horizontal, 16)
                .padding(.top, 16)
            }
            .scrollDismissesKeyboard(.interactively)
            .defaultScrollAnchor(.bottom)
            .contentMargins(.bottom, chatState.blankSize, for: .scrollContent)
            .safeAreaInset(edge: .bottom) {
                FloatingComposer()
                    .focused($isComposerFocused)
            }
        }
    }
}

// MARK: - Unified Message View
struct MessageView: View {
    let message: ChatMessage
    let index: Int
    @Environment(ChatState.self) private var state
    
    var body: some View {
        Group {
            switch message.role {
            case .user:
                UserMessageView(message: message, isFirstMessage: index == 0)
            case .assistant:
                AssistantMessageView(message: message, isFirstAssistantMessage: index == 1)
            case .optimisticPlaceholder:
                OptimisticPlaceholderView()
            }
        }
    }
}

// MARK: - Optimistic Placeholder (Typing Indicator)
struct OptimisticPlaceholderView: View {
    @State private var dotIndex = 0
    
    var body: some View {
        HStack {
            HStack(spacing: 4) {
                ForEach(0..<3) { i in
                    Circle()
                        .fill(.secondary)
                        .frame(width: 8, height: 8)
                        .scaleEffect(dotIndex == i ? 1.2 : 0.8)
                        .animation(
                            .easeInOut(duration: 0.4)
                            .repeatForever()
                            .delay(Double(i) * 0.15),
                            value: dotIndex
                        )
                }
            }
            .padding(.horizontal, 16)
            .padding(.vertical, 12)
            .glassEffect()
            
            Spacer()
        }
        .onAppear {
            dotIndex = 1
        }
    }
}

#Preview {
    V0StyleChatView()
}
```

---

## Key Takeaways

| React Native Pattern | SwiftUI iOS 26 Equivalent | Min iOS Version |
|---------------------|---------------------------|-----------------|
| Context Providers | `@Observable` + `.environment()` | iOS 17+ |
| Reanimated shared values | `@State`, `withAnimation` | iOS 13+ |
| LegendList | `LazyVStack` + `ScrollViewReader` | iOS 14+ |
| useAnimatedReaction | `.onChange(of:)` modifier | iOS 17+ |
| contentInset | `.contentMargins()`, `.safeAreaPadding()` | iOS 17+ |
| KeyboardStickyView | `.safeAreaInset(edge:)` | iOS 15+ |
| @callstack/liquid-glass | Native `.glassEffect()` | **iOS 26+** |
| Zeego menus | Native `Menu`, `contextMenu` | iOS 14+ |
| useFirstMessageAnimation | `PhaseAnimator`, `@Animatable` | iOS 17+ |
| FadeInStaggered pool | Actor-based animation pool | iOS 13+ |
| Native patches | Built-in; no patching needed | - |
| Scroll anchor | `.defaultScrollAnchor(.bottom)` | iOS 18+ |
| Zoom transitions | `.navigationTransition(.zoom)` | iOS 18+ |

---

## Resources

- [WWDC 2025 - Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/)
- [WWDC 2025 - Meet Liquid Glass](https://developer.apple.com/videos/play/wwdc2025/219/)
- [Apple Liquid Glass Documentation](https://developer.apple.com/documentation/TechnologyOverviews/liquid-glass)
- [SwiftUI GlassEffectContainer](https://developer.apple.com/documentation/swiftui/glasseffectcontainer)
- [Landmarks Sample Code](https://developer.apple.com/documentation/swiftui/landmarks-building-an-app-with-liquid-glass)

---

*Document generated based on Vercel's v0 iOS engineering blog post and iOS 26 WWDC 2025 documentation.*
