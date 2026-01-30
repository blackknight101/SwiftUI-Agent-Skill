# SwiftUI Animation Patterns Reference

## Core Concepts

### How SwiftUI Animations Work

State changes are the only way to trigger view updates. By default, changes aren't animated, but SwiftUI provides mechanisms to animate them.

**Animation Process:**
1. State change triggers view tree re-evaluation
2. SwiftUI compares new view tree to current render tree
3. Animatable properties are identified
4. Timing curve generates progress values (0 to 1)
5. Values interpolate smoothly (~60 fps)

**Key Characteristics:**
- Animations are additive and cancelable
- Always start from current render tree state
- Blend smoothly when interrupted

---

## Implicit vs Explicit Animations

### 1. Implicit Animations

Use `.animation(_:value:)` to animate when a specific value changes.

```swift
// GOOD - uses value parameter for precise control
struct GoodImplicitAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .animation(.spring, value: isExpanded)
            .onTapGesture { isExpanded.toggle() }
    }
}

// BAD - deprecated animation without value (animates everything)
struct BadImplicitAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .animation(.spring)  // Deprecated! Animates all changes unexpectedly
            .onTapGesture { isExpanded.toggle() }
    }
}
```

### 2. Explicit Animations

Use `withAnimation` to wrap state changes that should be animated.

```swift
// GOOD - explicit animation for event-driven changes
struct GoodExplicitAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation(.spring) {
                    isExpanded.toggle()
                }
            }

            Rectangle()
                .frame(width: isExpanded ? 200 : 100, height: 50)
        }
    }
}

// BAD - state change without animation context
struct BadExplicitAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            Button("Toggle") {
                isExpanded.toggle()  // No animation - abrupt change
            }

            Rectangle()
                .frame(width: isExpanded ? 200 : 100, height: 50)
        }
    }
}
```

### 3. Animation Placement

```swift
// GOOD - animation placed after the properties it should animate
struct GoodAnimationPlacement: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .foregroundStyle(isExpanded ? .blue : .red)
            .animation(.default, value: isExpanded)  // Animates both frame and color
    }
}

// BAD - animation placed before properties (may not animate as expected)
struct BadAnimationPlacement: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .animation(.default, value: isExpanded)  // Too early!
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .foregroundStyle(isExpanded ? .blue : .red)
    }
}
```

### 4. Selective Animation

```swift
// GOOD - animate only specific properties
struct GoodSelectiveAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .animation(.spring, value: isExpanded)  // Animate size
            .foregroundStyle(isExpanded ? .blue : .red)
            .animation(nil, value: isExpanded)  // Don't animate color
    }
}

// iOS 17+ scoped animation
struct GoodScopedAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .foregroundStyle(isExpanded ? .blue : .red)  // Not animated
            .animation(.spring) {
                $0.frame(width: isExpanded ? 200 : 100, height: 50)  // Only this is animated
            }
    }
}

// BAD - animating everything when only some properties should animate
struct BadSelectiveAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .foregroundStyle(isExpanded ? .blue : .red)
            .animation(.spring, value: isExpanded)  // Animates both - maybe unintended
    }
}
```

---

## Transitions

### 1. Basic Transitions

```swift
// GOOD - transition with animation outside conditional
struct GoodTransition: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            Button("Toggle") {
                showDetail.toggle()
            }

            if showDetail {
                DetailView()
                    .transition(.slide)
            }
        }
        .animation(.spring, value: showDetail)  // Animation outside conditional
    }
}

// GOOD - explicit animation for transitions
struct GoodExplicitTransition: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation(.spring) {
                    showDetail.toggle()
                }
            }

            if showDetail {
                DetailView()
                    .transition(.scale.combined(with: .opacity))
            }
        }
    }
}

// BAD - animation inside conditional (gets removed with the view!)
struct BadTransition: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            Button("Toggle") {
                showDetail.toggle()
            }

            if showDetail {
                DetailView()
                    .transition(.slide)
                    .animation(.spring, value: showDetail)  // Won't work on removal!
            }
        }
    }
}

// BAD - transition without any animation
struct BadNoAnimationTransition: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            Button("Toggle") {
                showDetail.toggle()  // No animation context
            }

            if showDetail {
                DetailView()
                    .transition(.slide)  // Transition ignored - view just appears/disappears
            }
        }
    }
}
```

### 2. Asymmetric Transitions

```swift
// GOOD - different animations for insertion vs removal
struct GoodAsymmetricTransition: View {
    @State private var showCard = false

    var body: some View {
        VStack {
            Button("Toggle Card") {
                withAnimation(.spring) {
                    showCard.toggle()
                }
            }

            if showCard {
                CardView()
                    .transition(
                        .asymmetric(
                            insertion: .scale.combined(with: .opacity),
                            removal: .move(edge: .bottom).combined(with: .opacity)
                        )
                    )
            }
        }
    }
}

// BAD - using same transition when different behaviors are needed
struct BadSymmetricTransition: View {
    @State private var showCard = false

    var body: some View {
        VStack {
            Button("Toggle Card") {
                withAnimation {
                    showCard.toggle()
                }
            }

            if showCard {
                CardView()
                    .transition(.slide)  // Same animation both ways - may feel awkward
            }
        }
    }
}
```

### 3. Custom Transitions

```swift
// GOOD - reusable custom transition (iOS 17+)
struct BlurTransition: Transition {
    var radius: CGFloat

    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .blur(radius: phase.isIdentity ? 0 : radius)
            .opacity(phase.isIdentity ? 1 : 0)
    }
}

extension AnyTransition {
    static var blur: AnyTransition {
        .modifier(
            active: BlurModifier(radius: 10),
            identity: BlurModifier(radius: 0)
        )
    }
}

struct GoodCustomTransition: View {
    @State private var showContent = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation(.easeInOut(duration: 0.5)) {
                    showContent.toggle()
                }
            }

            if showContent {
                ContentView()
                    .transition(BlurTransition(radius: 10))
            }
        }
    }
}

// BAD - inline complex transition logic
struct BadCustomTransition: View {
    @State private var showContent = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation {
                    showContent.toggle()
                }
            }

            if showContent {
                ContentView()
                    .blur(radius: showContent ? 0 : 10)  // Not a transition - won't animate on removal
                    .opacity(showContent ? 1 : 0)
            }
        }
    }
}
```

---

## Property Animations vs Transitions

```swift
// GOOD - property animation for existing view changes
struct GoodPropertyAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        // Same Rectangle exists before and after - property animation
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .animation(.spring, value: isExpanded)
            .onTapGesture { isExpanded.toggle() }
    }
}

// GOOD - transition for view insertion/removal
struct GoodTransitionUsage: View {
    @State private var showLarge = false

    var body: some View {
        VStack {
            Button("Toggle") {
                withAnimation {
                    showLarge.toggle()
                }
            }

            // Different views - use transition
            if showLarge {
                LargeRectangle()
                    .transition(.scale)
            } else {
                SmallRectangle()
                    .transition(.scale)
            }
        }
    }
}

// BAD - expecting property animation when identity changes
struct BadIdentityChange: View {
    @State private var isExpanded = false

    var body: some View {
        // Different branches = different identities = transition, not property animation
        if isExpanded {
            Rectangle()
                .frame(width: 200, height: 50)
                .animation(.spring, value: isExpanded)  // Won't smoothly animate!
        } else {
            Rectangle()
                .frame(width: 100, height: 50)
                .animation(.spring, value: isExpanded)
        }
    }
}
```

---

## The Animatable Protocol

### 1. Custom Animatable Modifier

```swift
// GOOD - proper Animatable implementation
struct ShakeModifier: ViewModifier, Animatable {
    var shakeCount: Double

    var animatableData: Double {
        get { shakeCount }
        set { shakeCount = newValue }
    }

    func body(content: Content) -> some View {
        content
            .offset(x: sin(shakeCount * .pi * 2) * 10)
    }
}

extension View {
    func shake(count: Int) -> some View {
        modifier(ShakeModifier(shakeCount: Double(count)))
    }
}

struct GoodShakeAnimation: View {
    @State private var shakeCount = 0

    var body: some View {
        Button("Shake Me") {
            shakeCount += 3
        }
        .shake(count: shakeCount)
        .animation(.default, value: shakeCount)
    }
}

// BAD - missing animatableData implementation
struct BadShakeModifier: ViewModifier {
    var shakeCount: Double

    // Missing animatableData! Uses default EmptyAnimatableData

    func body(content: Content) -> some View {
        content
            .offset(x: sin(shakeCount * .pi * 2) * 10)
    }
}

struct BadShakeAnimation: View {
    @State private var shakeCount = 0

    var body: some View {
        Button("Shake Me") {
            shakeCount += 3
        }
        .modifier(BadShakeModifier(shakeCount: Double(shakeCount)))
        .animation(.default, value: shakeCount)  // Won't animate - jumps to final value
    }
}
```

### 2. Multiple Animatable Properties

```swift
// GOOD - using AnimatablePair for multiple properties
struct ComplexAnimationModifier: ViewModifier, Animatable {
    var scale: CGFloat
    var rotation: Double

    var animatableData: AnimatablePair<CGFloat, Double> {
        get { AnimatablePair(scale, rotation) }
        set {
            scale = newValue.first
            rotation = newValue.second
        }
    }

    func body(content: Content) -> some View {
        content
            .scaleEffect(scale)
            .rotationEffect(.degrees(rotation))
    }
}

// GOOD - nested AnimatablePair for 3+ properties
struct ThreePropertyModifier: ViewModifier, Animatable {
    var x: CGFloat
    var y: CGFloat
    var rotation: Double

    var animatableData: AnimatablePair<AnimatablePair<CGFloat, CGFloat>, Double> {
        get { AnimatablePair(AnimatablePair(x, y), rotation) }
        set {
            x = newValue.first.first
            y = newValue.first.second
            rotation = newValue.second
        }
    }

    func body(content: Content) -> some View {
        content
            .offset(x: x, y: y)
            .rotationEffect(.degrees(rotation))
    }
}

// BAD - separate state for each property (can't coordinate animation)
struct BadMultiPropertyAnimation: View {
    @State private var scale: CGFloat = 1
    @State private var rotation: Double = 0

    var body: some View {
        Rectangle()
            .scaleEffect(scale)
            .rotationEffect(.degrees(rotation))
            .animation(.spring, value: scale)
            .animation(.spring, value: rotation)
            .onTapGesture {
                // These animate separately, potentially out of sync
                scale = scale == 1 ? 1.5 : 1
                rotation = rotation == 0 ? 45 : 0
            }
    }
}
```

---

## Animation Performance

### 1. Prefer Transforms Over Layout

```swift
// GOOD - animating transforms (GPU accelerated)
struct GoodPerformanceAnimation: View {
    @State private var isActive = false

    var body: some View {
        Rectangle()
            .frame(width: 100, height: 100)
            .scaleEffect(isActive ? 1.5 : 1.0)  // Transform - fast
            .offset(x: isActive ? 50 : 0)  // Transform - fast
            .rotationEffect(.degrees(isActive ? 45 : 0))  // Transform - fast
            .animation(.spring, value: isActive)
            .onTapGesture { isActive.toggle() }
    }
}

// BAD - animating layout properties (triggers layout passes)
struct BadPerformanceAnimation: View {
    @State private var isActive = false

    var body: some View {
        Rectangle()
            .frame(
                width: isActive ? 150 : 100,  // Layout change - expensive
                height: isActive ? 150 : 100
            )
            .padding(isActive ? 50 : 0)  // Layout change - expensive
            .animation(.spring, value: isActive)
            .onTapGesture { isActive.toggle() }
    }
}
```

### 2. Narrow Animation Scope

```swift
// GOOD - animation scoped to specific subview
struct GoodScopedAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            HeaderView()  // Not affected by animation

            ExpandableContent(isExpanded: isExpanded)
                .animation(.spring, value: isExpanded)  // Only this animates

            FooterView()  // Not affected by animation
        }
    }
}

// BAD - animation at root affects entire tree
struct BadBroadAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            HeaderView()
            ExpandableContent(isExpanded: isExpanded)
            FooterView()
        }
        .animation(.spring, value: isExpanded)  // Animates everything unnecessarily
    }
}
```

### 3. Avoid Animation in Hot Paths

```swift
// GOOD - gate animations by threshold
struct GoodScrollAnimation: View {
    @State private var showTitle = false

    var body: some View {
        ScrollView {
            // Content
        }
        .onPreferenceChange(ScrollOffsetKey.self) { offset in
            let shouldShow = offset.y < -50
            if shouldShow != showTitle {  // Only update when crossing threshold
                withAnimation(.easeOut(duration: 0.2)) {
                    showTitle = shouldShow
                }
            }
        }
    }
}

// BAD - animating on every scroll position change
struct BadScrollAnimation: View {
    @State private var offset: CGFloat = 0

    var body: some View {
        ScrollView {
            // Content
        }
        .onPreferenceChange(ScrollOffsetKey.self) { newOffset in
            withAnimation {  // Fires constantly during scroll!
                offset = newOffset.y
            }
        }
    }
}
```

---

## Phase Animations (iOS 17+)

```swift
// GOOD - phase animator for multi-step sequences
struct GoodPhaseAnimation: View {
    @State private var trigger = 0

    var body: some View {
        Button("Animate") {
            trigger += 1
        }
        .phaseAnimator(
            [0.0, -10.0, 10.0, -5.0, 5.0, 0.0],
            trigger: trigger
        ) { content, offset in
            content.offset(x: offset)
        } animation: { phase in
            switch phase {
            case 0: .smooth
            default: .snappy
            }
        }
    }
}

// GOOD - using enum phases for clarity
enum BouncePhase: CaseIterable {
    case initial, up, down, settle

    var scale: CGFloat {
        switch self {
        case .initial: 1.0
        case .up: 1.2
        case .down: 0.9
        case .settle: 1.0
        }
    }
}

struct GoodEnumPhaseAnimation: View {
    @State private var trigger = 0

    var body: some View {
        Circle()
            .frame(width: 50, height: 50)
            .phaseAnimator(BouncePhase.allCases, trigger: trigger) { content, phase in
                content.scaleEffect(phase.scale)
            }
            .onTapGesture { trigger += 1 }
    }
}

// BAD - using custom Animatable when phaseAnimator would be simpler
struct BadManualMultiStep: View {
    @State private var animationProgress: Double = 0

    var body: some View {
        Button("Animate") {
            // Complex manual animation sequencing
            withAnimation(.easeOut(duration: 0.1)) {
                animationProgress = 0.25
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                withAnimation(.easeInOut(duration: 0.1)) {
                    animationProgress = 0.5
                }
            }
            // ... more manual sequencing
        }
        .offset(x: sin(animationProgress * .pi * 4) * 20)
    }
}
```

---

## Keyframe Animations (iOS 17+)

```swift
// GOOD - keyframe animation for precise timing control
struct GoodKeyframeAnimation: View {
    @State private var trigger = 0

    var body: some View {
        Button("Bounce") {
            trigger += 1
        }
        .keyframeAnimator(
            initialValue: AnimationValues(),
            trigger: trigger
        ) { content, value in
            content
                .scaleEffect(value.scale)
                .offset(y: value.verticalOffset)
        } keyframes: { _ in
            KeyframeTrack(\.scale) {
                SpringKeyframe(1.2, duration: 0.15)
                SpringKeyframe(0.9, duration: 0.1)
                SpringKeyframe(1.0, duration: 0.15)
            }

            KeyframeTrack(\.verticalOffset) {
                LinearKeyframe(-20, duration: 0.15)
                LinearKeyframe(0, duration: 0.25)
            }
        }
    }
}

struct AnimationValues {
    var scale: CGFloat = 1.0
    var verticalOffset: CGFloat = 0
}

// GOOD - multiple synchronized tracks
struct GoodMultiTrackKeyframe: View {
    @State private var trigger = 0

    var body: some View {
        Image(systemName: "bell.fill")
            .font(.largeTitle)
            .keyframeAnimator(
                initialValue: BellAnimation(),
                trigger: trigger
            ) { content, value in
                content
                    .rotationEffect(.degrees(value.rotation))
                    .scaleEffect(value.scale)
            } keyframes: { _ in
                KeyframeTrack(\.rotation) {
                    CubicKeyframe(15, duration: 0.1)
                    CubicKeyframe(-15, duration: 0.1)
                    CubicKeyframe(10, duration: 0.1)
                    CubicKeyframe(-10, duration: 0.1)
                    CubicKeyframe(0, duration: 0.1)
                }

                KeyframeTrack(\.scale) {
                    CubicKeyframe(1.1, duration: 0.25)
                    CubicKeyframe(1.0, duration: 0.25)
                }
            }
            .onTapGesture { trigger += 1 }
    }
}

struct BellAnimation {
    var rotation: Double = 0
    var scale: CGFloat = 1.0
}

// BAD - manual timer-based animation
struct BadManualKeyframe: View {
    @State private var rotation: Double = 0
    @State private var scale: CGFloat = 1.0

    var body: some View {
        Image(systemName: "bell.fill")
            .rotationEffect(.degrees(rotation))
            .scaleEffect(scale)
            .onTapGesture {
                // Manual timing - error prone and hard to maintain
                withAnimation(.easeOut(duration: 0.1)) { rotation = 15 }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                    withAnimation(.easeOut(duration: 0.1)) { rotation = -15 }
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
                    withAnimation(.easeOut(duration: 0.1)) { rotation = 0 }
                }
                // ... more manual timing
            }
    }
}
```

---

## Animation Completion Handlers (iOS 17+)

```swift
// GOOD - completion with withAnimation
struct GoodCompletionHandler: View {
    @State private var isExpanded = false
    @State private var showNextStep = false

    var body: some View {
        VStack {
            Rectangle()
                .frame(width: isExpanded ? 200 : 100, height: 50)

            if showNextStep {
                Text("Animation Complete!")
            }
        }
        .onTapGesture {
            withAnimation(.spring) {
                isExpanded.toggle()
            } completion: {
                showNextStep = true
            }
        }
    }
}

// GOOD - completion with transaction (reexecutes properly)
struct GoodTransactionCompletion: View {
    @State private var bounceCount = 0
    @State private var message = ""

    var body: some View {
        VStack {
            Circle()
                .frame(width: 50, height: 50)
                .scaleEffect(bounceCount % 2 == 0 ? 1.0 : 1.2)
                .transaction(value: bounceCount) { transaction in
                    transaction.animation = .spring
                    transaction.addAnimationCompletion {
                        message = "Bounce \(bounceCount) complete"
                    }
                }

            Text(message)

            Button("Bounce") {
                bounceCount += 1
            }
        }
    }
}

// BAD - completion handler without value parameter (only fires once)
struct BadCompletionHandler: View {
    @State private var bounceCount = 0
    @State private var completionCount = 0

    var body: some View {
        VStack {
            Circle()
                .frame(width: 50, height: 50)
                .scaleEffect(bounceCount % 2 == 0 ? 1.0 : 1.2)
                .animation(.spring, value: bounceCount)
                .transaction { transaction in  // No value parameter!
                    transaction.addAnimationCompletion {
                        completionCount += 1  // Only fires once, ever
                    }
                }

            Text("Completions: \(completionCount)")

            Button("Bounce") {
                bounceCount += 1
            }
        }
    }
}
```

---

## Timing Curves

```swift
// GOOD - appropriate timing curve for the interaction
struct GoodTimingCurves: View {
    @State private var isActive = false

    var body: some View {
        VStack(spacing: 20) {
            // Quick response for interactive elements
            Button("Tap Me") {
                withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) {
                    isActive.toggle()
                }
            }
            .scaleEffect(isActive ? 0.95 : 1.0)

            // Smooth for appearance changes
            Rectangle()
                .foregroundStyle(isActive ? .blue : .gray)
                .animation(.easeInOut(duration: 0.25), value: isActive)

            // Bouncy for playful feedback
            Circle()
                .scaleEffect(isActive ? 1.2 : 1.0)
                .animation(.bouncy(duration: 0.4), value: isActive)
        }
    }
}

// BAD - inappropriate timing curves
struct BadTimingCurves: View {
    @State private var isActive = false

    var body: some View {
        VStack(spacing: 20) {
            // Too slow for button feedback
            Button("Tap Me") {
                withAnimation(.easeInOut(duration: 1.0)) {  // Way too slow!
                    isActive.toggle()
                }
            }
            .scaleEffect(isActive ? 0.95 : 1.0)

            // Linear feels robotic for UI
            Rectangle()
                .foregroundStyle(isActive ? .blue : .gray)
                .animation(.linear(duration: 0.5), value: isActive)  // Feels mechanical

            // No easing on size changes feels abrupt
            Circle()
                .scaleEffect(isActive ? 1.2 : 1.0)
                .animation(.linear(duration: 0.1), value: isActive)  // Too abrupt
        }
    }
}
```

---

## Disabling Animations

```swift
// GOOD - selectively disable animations
struct GoodDisableAnimation: View {
    @State private var count = 0

    var body: some View {
        VStack {
            // This should animate
            Circle()
                .frame(width: CGFloat(count * 10 + 50))
                .animation(.spring, value: count)

            // This should NOT animate (immediate feedback)
            Text("Count: \(count)")
                .transaction { $0.animation = nil }

            Button("Increment") {
                count += 1
            }
        }
    }
}

// GOOD - disable animations from parent context
struct GoodDisableParentAnimation: View {
    @State private var isLoading = false

    var body: some View {
        VStack {
            if isLoading {
                ProgressView()
                    .transition(.opacity)
            }

            DataView()
                .transaction { transaction in
                    transaction.disablesAnimations = true  // No animations for data updates
                }
        }
        .animation(.default, value: isLoading)
    }
}

// BAD - fighting animation system with zero duration
struct BadDisableAnimation: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
                .animation(.linear(duration: 0), value: count)  // Hacky way to disable

            Button("Increment") {
                count += 1
            }
        }
    }
}
```

---

## Debugging Animations

```swift
// GOOD - debug modifier to inspect animation values
struct AnimationDebugModifier: ViewModifier, Animatable {
    var value: Double
    let label: String

    var animatableData: Double {
        get { value }
        set {
            value = newValue
            #if DEBUG
            print("[\(label)] Animation value: \(newValue)")
            #endif
        }
    }

    func body(content: Content) -> some View {
        content.opacity(value)
    }
}

// GOOD - slow down animation for inspection
struct GoodDebugAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            #if DEBUG
            .animation(.linear(duration: 3.0).speed(0.2), value: isExpanded)  // 15 second animation
            #else
            .animation(.spring, value: isExpanded)
            #endif
            .onTapGesture { isExpanded.toggle() }
    }
}

// BAD - no way to debug animation issues
struct BadNoDebugAnimation: View {
    @State private var isExpanded = false

    var body: some View {
        Rectangle()
            .frame(width: isExpanded ? 200 : 100, height: 50)
            .animation(.spring, value: isExpanded)  // Animation not working? No way to tell why
            .onTapGesture { isExpanded.toggle() }
    }
}
```

---

## Summary Checklist

### Do
- [ ] Use `.animation(_:value:)` with value parameter
- [ ] Use `withAnimation` for event-driven animations
- [ ] Place transitions outside conditional structures
- [ ] Implement `animatableData` explicitly for custom Animatable types
- [ ] Prefer transforms (`scale`, `offset`, `rotation`) over layout changes
- [ ] Use `.phaseAnimator` for multi-step sequences (iOS 17+)
- [ ] Use `.keyframeAnimator` for precise timing (iOS 17+)
- [ ] Use `.transaction(value:)` for completion handlers that should refire
- [ ] Scope animations narrowly to affected views
- [ ] Choose appropriate timing curves for the interaction type

### Don't
- [ ] Use deprecated `.animation(_:)` without value parameter
- [ ] Put animation modifiers inside conditionals for transitions
- [ ] Forget `animatableData` implementation (silent failure)
- [ ] Animate layout properties in performance-critical paths
- [ ] Use manual DispatchQueue timing for sequenced animations
- [ ] Apply broad animations at root view level
- [ ] Use linear timing for UI interactions (feels robotic)
- [ ] Animate on every frame in scroll handlers
