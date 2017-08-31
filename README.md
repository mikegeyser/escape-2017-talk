# Building a twitter wall in 15 minutes, and learning something along the way
# Set up the server

### server/index.js
> # _express
``` javascript
var app = require('express')();
var server = require('http').createServer(app);
var io = require('socket.io')(server);

server.listen(3000);
```

> # _ping
```js
setInterval(() => io.emit('ping'), 3000);
```

# Create a ping component

```bash
> ng generate component ping
```

### app/src/app/app.component.html
```html
<app-ping></app-ping>
```

### app/src/app/ping/ping.component.ts
```ts 
declare let io: any; 
```
```ts 
shouldPing = false;
```
> # _ping1
```ts
let socket = io('http://localhost:3000/');
socket.on('ping', (ping) => {
  this.shouldPing = true;
  setTimeout(() => this.shouldPing = false, 1000);
});
```

### app/src/app/ping/ping.component.html
> # _ping
```html
<div class="dot" [class.ping]="shouldPing"></div>
```

# Change the ping component

### app/src/app/ping/ping.component.ts
> # _ping_imports
```ts
import { BehaviorSubject } from 'rxjs/BehaviorSubject';
import 'rxjs/add/operator/debounceTime';
```

> # _ping_subject
```ts
shouldPing = new BehaviorSubject(false);
```

> # _ping_2
```ts
socket.on('ping', () => this.shouldPing.next(true));

this.shouldPing
  .debounceTime(1000)
  .subscribe(x => this.shouldPing.next(false));
```

### app/src/app/ping/ping.component.html
```html
"shouldPing | async"
```

# Create the stream service
```bash
> ng generate service stream
```

### app/src/app/app.module.ts
```ts
import { StreamService } from './stream.service';
```
```ts
providers: [StreamService],
```

### app/src/app/stream.service.ts
> # _service_1
```ts
import { Injectable, OnInit } from '@angular/core';
import { BehaviorSubject } from 'rxjs/BehaviorSubject';
import 'rxjs/add/operator/debounceTime';

declare let io: any;

@Injectable()
export class StreamService {
  ping = new BehaviorSubject<boolean>(false);

  constructor() {
    let socket = io('http://localhost:3000/');

    socket.on('ping', () => this.ping.next(true));

    this.ping
      .debounceTime(1000)
      .subscribe(x => this.ping.next(false));
  }
}
```

### app/src/app/ping/ping.component.ts
> # _ping_3
```ts
import { Component, OnInit } from '@angular/core';
import { StreamService } from '../stream.service';

@Component({
  selector: 'app-pulse',
  templateUrl: './pulse.component.html'
})
export class PingComponent {
  constructor(public stream: StreamService) { }
}
```
### app/src/app/ping/ping.component.html
```html
"stream.ping | async"
```

# Add twitter streaming to the server
### server/index.js
> # _twitter_config
```js
let Twit = require('twit');
let T = new Twit({
  'consumer_key': '{{redacted}}',
  'consumer_secret': '{{redacted}}',
  'access_token': '{{redacted}}',
  'access_token_secret': '{{redacted}}',
  'timeout_ms': 60000
});
```

> # _twitter_stream
```js
let stream = T.stream('statuses/filter', {
  track: [
    '@BBDSoftware',
    '@mikegeyser',
    '@bbddevplus',
    '#BBDEscape',
  ].join(','),
  language: 'en'
});
```

> # _tweet_emit
```js
stream.on('tweet', (status) => {
  console.log(status);
  io.emit('tweet', status);
});
```

> # _tweet_store
```js
let tweets = [];
stream.on('tweet', (status) => {
  console.log(status);
    tweets.unshift(status);
    
    if (tweets.length > 100)
      tweets.splice(100);
  io.emit('tweet', status);
});
```

> # _get_all
```js
var cors = require('cors');
app.use(cors())
app.get('/tweets', (request, response) => response.send(tweets));
```

# Add twitter streaming to the app service
### app/src/app/stream.service.ts
> # _service_tweet_import
```ts
import { Http } from '@angular/http';
import { Observable } from 'rxjs/Observable';
import 'rxjs/add/operator/mergeMap';
```

```ts
tweets = new BehaviorSubject([]);
```

```ts
constructor(http: Http) {
```

># _service_tweet_1
```ts
let get = http.get('http://localhost:3000/tweets').map(response => response.json());
```

> # _service_tweet_2
```ts
let emit = Observable.create(observer => {
  socket.on('tweet', (tweet) => {
    observer.next(tweet);
  });
});
```

> # _service_tweet_3
```ts
get.subscribe(tweets => this.tweets.next(tweets));
```

> # _service_tweet_4
```ts
emit.subscribe(tweet => {
  let tweets = this.tweets.value;
  tweets.unshift(tweet);

  if (tweets.length > 100)
    tweets.splice(100);

  this.tweets.next(tweets);
});
```

> # _service_tweet_5
```ts
let emit = (tweets) => Observable.create(observer => {
  observer.next(tweets);

  socket.on('tweet', (tweet) => {
    tweets.unshift(tweet);

    if (tweets.length > 100)
      tweets.splice(100);

    observer.next(tweets);
  });
});
```

> # _service_tweet_6
```ts
this.tweets = get.switchMap(x => emit(x));
```

# Add the service to the app component
### app/src/app/app.component.ts
```ts
import { StreamService } from './stream.service';
```

> # _app_1
```ts
constructor(public stream: StreamService) { }
```

# Add some initial markup to the app component
### app/src/app/app.component.html
> # _app_1
```ts
<div *ngFor="let tweet of stream.tweets | async">
  {{tweet.text}}
</div>
```

# Create a tweet component
```bash
> ng generate component tweet
```

### app/src/app/tweet/tweet.component.ts
```ts
import { Component, Input, OnInit } from '@angular/core';
```

```ts
@Input() tweet: any;
```

### app/src/app/tweet/tweet.component.html
> # _tweet
```html
<div class="EmbeddedTweet">
  <div class="EmbeddedTweet-tweet">
    <blockquote class="Tweet">
      <div class="Tweet-header">
        <div class="TweetAuthor">
          <span class="TweetAuthor-link Identity" href="">
            <span class="TweetAuthor-avatar Identity-avatar">
              <img class="Avatar" alt="" src="{{ tweet.user.profile_image_url }}">
            </span>
          <span class="TweetAuthor-name Identity-name">{{ tweet.user.name }}</span>
          <span class="TweetAuthor-screenName Identity-screenName">@{{tweet.user.screen_name }}
          </span>
          </span>
        </div>
      </div>
      <div class="Tweet-body">
        <p class="Tweet-text">{{ tweet.text }}</p>
        <div class="Tweet-metadata dateline">
          <span class="long-permalink" href="">
            <time>{{ tweet.created_at | date: 'short'}}</time>
          </span>
        </div>
      </div>
    </blockquote>
  </div>
</div>
```

### app/src/app/app.component.html
> # _app_2
```html
<app-tweet *ngFor="let tweet of stream.tweets | async" [tweet]="tweet">
</app-tweet>
```

# Add some flair (and show observable subscriptions)
### app/src/app/app.component.ts
> # _app_declare_masonry
```ts
declare let Masonry: any;
```
> # _app_new_masonry
```ts
var masonry = new Masonry('app-root', {
  itemSelector: '.EmbeddedTweet'
});
```

> # _app_masonry_subscribe
```ts
this.stream.tweets.debounceTime(1).subscribe(x => {
  masonry.reloadItems();
  masonry.layout();
});
```

# done!

