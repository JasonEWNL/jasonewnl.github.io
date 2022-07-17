---
layout: post
title: Compose ZhiHu Daily with Widgets in iOS 14
date: 2020-08-10
description: As a common project for both amateurs and professionals, ZhiHu Daily is a friendly approach to explain WidgetKit introduced in WWDC20.
tag:
- WidgetKit
---

> The complete project is available at [GitHub](https://github.com/JasonEWNL/ZhiHuDaily), which provides full encapsulation and optimization you will be interested. Also you might find it more vivid alongside the holistic structure during this trip.

## Origin

Since Apple announced WidgetKit in WWDC20, the tide of ideas have been flooding through my mind. Although some limitation like no interaction except deep link might quench the fire, it still has a great deal of stuff to dive into. Along with the stabilisation of API design like placeholder, it's a good time to fill the tank up! This time we will use ZhiHu Daily for explanation.

![ZhiHuDaily-Screenshot-1](../../assets/images/ZhiHuDaily-Screenshot-1.png)

## Preparation

WidgetKit requires Xcode 12 which is still in beta 4, and I strongly recommend you to make a backup or install on a test device.

## Getting Started

### Overlook

An important reason choosing ZhiHu Daily is that the API design is good enough to talk about the main idea while it's simple to explain. In this project we pick [the latest news](https://news-at.zhihu.com/api/4/news/latest) for a taste:

```json
{
    "date":"20200810",
    "stories": [...],
    "top_stories": [
        {
            "image_hue": "0xb3774e",
            "hint": "作者 / 地道风物",
            "url": "https://daily.zhihu.com/story/9726632",
            "image": "https://pic4.zhimg.com/v2-868f475c699a5c1e29be0cc3ef16a81b.jpg",
            "title": "为什么沈阳人那么爱吃鸡架？",
            "ga_prefix": "080707",
            "type": 0,
            "id": 9726632
        },
        ...
    ]
}
```

It might be different if you fetch it on some other day, but the bulk stay the same.

### Basis

The first thing after gaining an API is building a model accordingly, and later we use the `TopStory` for the main course:

```swift
struct News: Codable {
    let date: String
    let stories: [Story]
    let topStories: [TopStory]
}

struct TopStory: Codable {
    let imageHue, hint: String
    let url: String
    let image: String
    let title, gaPrefix: String
    let type, id: Int
}
```

Easily we can hire a manager for networking, whose `Network` returning a data-response pair is analogous to other framework:

```swift
struct NewsManager {
    static func newsGet(completion: @escaping (Result<News, Network.Failure>) -> Void) {
        Network.fetch("https://news-at.zhihu.com/api/4/news/latest") { result in
            switch result {
            case .success(let (data, response)):
                guard response.statusCode == 200 else {
                    completion(.failure(.requestFailed))
                    return
                }
                
                guard let news = try? JSONDecoder().decode(News.self, from: data) else {
                    completion(.failure(.decodeFailed))
                    return
                }
                
                completion(.success(news))
            case .failure(let error):
                completion(.failure(error))
            }
        }
    }
}
```

## Going Deep

### Create the Widget Target

After finishing basis of model, we can begin our widgets indeed. Firstly we create a "Widget Extension" via File - New - Target in the Xcode menu, and notice the "Include Configuration Intent" box is unchecked. Then you can see a group with a widget template and other files, and if you feel dazzled as I was at first time, you might expect to delete them all and start with a blank Swift source file. Also you would like to create separated files for following different part which can help you straighten up.

### Timeline Entry and Timeline Provider

Not like in general Swift, model in widgets is bound time. Since widgets might be updated for multiple times in a single timeline, `TimelineEntry` and `TimelineProvider` just naturally come up.

`TimelineEntry` contains normal model with a `date` property:

```swift
struct StoryEntry: TimelineEntry {
    let date: Date
    let storyPairs: [(TopStory, Image)]
}
```

At this time, even professional might wonder why set an array of tuple rather than `[TopStory]` directly, and I would say that it relates with the views in widgets can only be loaded before provided, which means that you can't refresh it after showing, Therefore, we need to fetch all image data in the provider:

```swift
let storyPairs = [
    (TopStory(title: "35 岁真的就很容易失业吗？", hint: "作者 / Sean Ye"), Image("9726467")),
    ...
]

struct StoryProvider: TimelineProvider {
    func placeholder(in context: Context) -> StoryEntry {
        StoryEntry(date: Date(), storyPairs: storyPairs)
    }

    func getSnapshot(in context: Context, completion: @escaping (StoryEntry) -> ()) {
        let entry = StoryEntry(date: Date(), storyPairs: storyPairs)
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<StoryEntry>) -> ()) {
        var entries: [StoryEntry] = []
        
        NewsManager.newsGet { result in
            switch result {
            case .success(let news):
                DispatchQueue.global().async {
                    let semaphore = DispatchSemaphore(value: 0)
                    
                    var storyPairs = [(TopStory, Image)]()
                    for story in news.topStories {
                        Network.batchImage(story.image) { result in
                            switch result {
                            case .success(let image):
                                storyPairs.append((story, image))
                            case .failure(let error):
                                print(error)
                            }
                            semaphore.signal()
                        }
                        semaphore.wait()
                    }
                    
                    let entry = StoryEntry(date: Date(), storyPairs: storyPairs)
                    entries.append(entry)
                    let timeline = Timeline(entries: entries, policy: .never)
                    DispatchQueue.main.async {
                        completion(timeline)
                    }
                }
            case .failure(let error):
                print(error)
            }
        }
    }
}
```

The first two function set the appearances for placeholder which at the loading state and snapshot which in the widgets' gallery, and we just use some static data here.

The last function is the big head. But don't worry, it's just like normal manager, we just get `New` and go through the `[TopStory]` alongside `Image`, it's definitely not any clean code but that's how it work.

### View and Widget

Like in normal SwiftUI, views are fully supported. However, the fascinating part is the deep link. Because of the difference of widget family, the medium and large sizes use `Link` while the small size just add `.widgetURL(_:)` modifier. Therefore we need set coordination in our model:

```swift
struct TopStory: Codable {
    ...
    
    var inAppURL: URL { URL(string: "zhihudaily://topStory/\(id)")! }
}
```

The final step is configuring the widget:

```swift
@main
struct StoryWidget: Widget {
    let kind: String = "StoryWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: StoryProvider()) { entry in
            StoryWidgetEntryView(entry: entry)
        }
        ...
    }
}
```

With `onOpenURL(perform:)` in the main app, we can implement the deep link which is entering specific news view in our case.

![ZhiHuDaily-Screenshot-2](../../assets/images/ZhiHuDaily-Screenshot-2.png)

## Conclusion

WidgetKit is a gold mine which is still under development, and I can't wait to see your inspiration this fall. Hope you get an overview of WidgetKit through this post!
