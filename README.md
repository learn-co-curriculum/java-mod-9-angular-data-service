# Data Service

## Learning Goals

- Use data service in Angular.

## Data Service in Angular

As we saw in the previous section, a service can have methods and variables that
can be accessible by any component that properly defines that service as a
dependency, while letting Angular manage the lifecycle of the service.

For example, we can expand the `MessagingDataService` we created earlier by
adding our `userMessages` and `senderMessages` arrays to it:

```typescript
import { Injectable, EventEmitter } from "@angular/core";
import { LoggingService } from "./logging.service";
import { Message } from "./message.model";

@Injectable()
export class MessagingDataService {
  private senderMessages: Message[] = [
    {
      sender: { firstName: "Ludovic", isOnline: true },
      text: "Message from Ludovic",
      conversationId: 1,
      sequenceNumber: 0,
    },
    {
      sender: { firstName: "Jessica" },
      text: "Message from Jessica",
      conversationId: 1,
      sequenceNumber: 1,
    },
  ];

  private userMessages: Message[] = [
    {
      sender: { firstName: "Aurelie" },
      text: "Message from Aurelie",
      conversationId: 1,
      sequenceNumber: 2,
    },
  ];

  getSenderMessages() {
    return this.senderMessages.slice();
  }

  getUserMessages() {
    return this.userMessages.slice();
  }

  constructor(private loggingSvce: LoggingService) {
    loggingSvce.log("Messaging Data Service constructor completed");
  }
}
```

Let's go over a couple of notable things about this code:

1. The `userMessages` and `senderMessages` variables are private
2. The `getUserMessages()` and `getSenderMessages()` functions are provided for
   clients of this service class to be able to gain access to those private
   members
3. However, you will note that those functions do not simply return the
   corresponding array. Instead they call the `slice()` function on those
   arrays. This is to ensure that the clients of these functions do not get a
   reference to the internal array, but instead get a copy of that array.
   Returning a reference would expose our service class to changes to its
   internal data structures from other classes and components, which is an
   anti-pattern.

> An anti-pattern is a pattern in your code that may not cause issues (i.e.
> bugs) right away, but that is known to lead to issues in later stages. Often
> times, anti-patterns are fairly easy to spot at the moment that they are
> created, but are hard to find later, when they are buried deep in an aging
> code base, and they start causing issues that are difficult to debug. The
> anti-pattern we used here is one where a class' data could be available
> outside of itself to be modified by other components of your system that do
> not (and should) know about the inner workings of your class.

Now that the `userMessages` and `senderMessages` arrays are given values in the
`MessagingDataService`, we no longer need to give them values in the
`ConversationThreadComponent`. Instead, we're going to keep the definition
there, so they're accessible in the corresponding view, but we're going to get
their actual value from our service:

```typescript
import { Component, OnInit } from "@angular/core";
import { Message } from "src/app/message.model";
import { MessagingDataService } from "src/app/messaging-data.service";

@Component({
  selector: "app-conversation-thread-component",
  templateUrl: "./conversation-thread-component.component.html",
  styleUrls: ["./conversation-thread-component.component.css"],
})
export class ConversationThreadComponentComponent implements OnInit {
  senderMessages: Message[];
  userMessages: Message[];

  constructor(private messagingSvce: MessagingDataService) {}

  ngOnInit(): void {
    this.senderMessages = this.messagingSvce.getSenderMessages();
    this.userMessages = this.messagingSvce.getUserMessages();
  }
}
```

Let's break down this code:

1. We import the class for our `MessagingDataService`
2. We import the `OnInit` interface from Angular in order to add a `ngOnInit()`
   lifecycle function to our component:
   1. The `ngOnInit()` function is not required for all components, but whenever
      a component has actual initialization work to perform, it is recommended
      to perform it in `ngOnInit()` instead of in the constructor, as that gives
      Angular more flexibility on when to take on the corresponding work.
3. We ask Angular to inject the `MessagingDataService` by declaring it in our
   constructor
   1. The `MessagingDataService` is available to be injected here because we
      declared it as a `provider` in our module class.
4. We declare `senderMessages` and `userMessages` as we did before, but we no
   longer assign them values here
5. We use the `messagingSvce` service to get the value for both message arrays

### Cross-component communication with Emitters

Now that we have our data in our service, and are accessing it from our
component, we have successfully put our data somewhere where other components,
including ours, are able to access it.

We do not currently, however, have the ability to make changes to this data in
our service and make any of the components that use it aware of those changes.

To make this problem more apparent, let's add an event handler to our
`send-message-component.component.html` view:

```html
<div class="row">
  <div class="col-10 p-3">
    <input
      type="input"
      class="form-control"
      placeholder="Type a message"
      [(ngModel)]="messageString"
    />
  </div>
  <div class="col-2 p-3">
    <div class="float-end">
      <button class="btn btn-primary" (click)="onSendMessage()">Send</button>
    </div>
  </div>
</div>
```

We then add the `onSendMessage()` function to our
`send-message-component.component.ts` controller:

```typescript
onSendMessage() {
    this.loggingSvce.log("Send following message: ");
    this.loggingSvce.log(this.messageString);
}
```

So now our application knows when the `Send` button is clicked, and calls the
right method is the right controller. In that method, we have access to the text
in the input box, thanks for the `ngModel` 2-way data binding we set up earlier.

But since the Send Message component does not own the list of messages displayed
in the Conversation Thread component, how can we tell that component that we now
have a new message that should be shown in that view.

This is what "Event Emitters" are for. We can register an Event Emitter in our
service, and use it to tell any component who might be interested when specific
service data members are updated. It's important to understand the direction in
which this needs to be set up: the service (`MessagingDataService`) is now
responsible for the messaging data. Therefore it should be a) the place where
this data is updated and b) the place that is responsible for telling other
components that this data is updated.

It might be tempting to take the more direct approach of having the component
own its data, which can work in very simple use cases, but that approach does
not scale when you have even a slightly more complex application, like the one
we're building here, where data has to be shared between components.

Let's add an `EventEmitter` to our `MessagingDataService`:

1. First, we need to import `EventEmitter` from Angular's core library:
   `import { Injectable, EventEmitter } from "@angular/core";`
2. Then we need to create an instance of `EventEmitter`:
   `userMessagesChanged = new EventEmitter<Message[]>();`
3. Note that `EventEmitter` is a generic, which means we can specify the type of
   events that we expect it to emit. Here, we expect an array of `Message`
   objects to be emitted
4. Since we want our `MessagingDataService` to be responsible for updates to the
   messages, we add a function that any component can call to add a message, in
   this case a `userMessage`:

```
addUserMessage(newMessage: Message) {
    this.userMessages.push(newMessage);
    this.userMessagesChanged.emit(this.userMessages.slice());
}
```

5. In this new function, we push the new message to our existing `userMessage`
   collection, and then we call the `emit()` function on the `EventEmitter` we
   created earlier. This is what will let any subscribers (we'll see how to set
   that up next) know that a change was made in the service.

The updated `MessagingDataService` looks like this:

```typescript
import { Injectable, EventEmitter } from "@angular/core";
import { LoggingService } from "./logging.service";
import { Message } from "./message.model";

@Injectable()
export class MessagingDataService {
  private senderMessages: Message[] = [
    {
      sender: { firstName: "Ludovic", isOnline: true },
      text: "Message from Ludovic",
      conversationId: 1,
      sequenceNumber: 0,
    },
    {
      sender: { firstName: "Jessica" },
      text: "Message from Jessica",
      conversationId: 1,
      sequenceNumber: 1,
    },
  ];

  private userMessages: Message[] = [
    {
      sender: { firstName: "Aurelie" },
      text: "Message from Aurelie",
      conversationId: 1,
      sequenceNumber: 2,
    },
  ];

  userMessagesChanged = new EventEmitter<Message[]>();

  getSenderMessages() {
    return this.senderMessages.slice();
  }

  getUserMessages() {
    return this.userMessages.slice();
  }

  addUserMessage(newMessage: Message) {
    this.userMessages.push(newMessage);
    this.userMessagesChanged.emit(this.userMessages.slice());
  }

  constructor(private loggingSvce: LoggingService) {
    loggingSvce.log("Messaging Data Service constructor completed");
  }
}
```

Now that we have emitted an event, let's look at the "subscription" side of it.

Events are a pattern of communication where instead of needing a direct call
between a caller and a callee, the communication is set so that one side of the
communication broadcasts an event, in this case this is the service through the
event emitter, and the other side can be 0, 1 or more "subscribers", who
register to be notified of events, and then can decide to consume those events
or ignore them altogether.

One of the main advantages of this pattern is that the emitter is decoupled from
the consumers/subscribers. In other words, the emitter does not need to know,
and doesn't care, who the subscribers are. It's responsible for producing the
event, and as long as it does that properly, its contract with the subscribers
is satisfied and their work can start from there.

In our case, the Conversation Thread component is going to be one of the
consumers of the events emitted by the Messaging Data Service. Here is the code
we need to add to the `ngOnInit()` function to set this up:

```typescript
this.messagingSvce.userMessagesChanged.subscribe((messages: Message[]) => {
  console.log("********** messages have changed");
  this.userMessages = messages;
});
```

Let's break it down:

1. As before, we have access to the `messagingSvce` that Angular injected for us
2. We access the event emitter we set up in the previous step
3. We call its `subscribe()` function
4. That function takes 2 parameters:
   1. One is the object that we expect to have emitted from the event - in this
      case, it's the collection of `Message` objects that represents the updated
      list
   2. The other is a function where we can take the appropriate action as a
      result of the event
5. In this case, the action is to update our local (component) copy of the
   `userMessages`, which will immediately update the view, since we have already
   set up data binding.

This is the updated `conversation-thread-component.component.ts`:

```typescript
import { Component, OnInit } from "@angular/core";
import { Message } from "src/app/message.model";
import { MessagingDataService } from "src/app/messaging-data.service";

@Component({
  selector: "app-conversation-thread-component",
  templateUrl: "./conversation-thread-component.component.html",
  styleUrls: ["./conversation-thread-component.component.css"],
})
export class ConversationThreadComponentComponent implements OnInit {
  senderMessages: Message[];
  userMessages: Message[];

  constructor(private messagingSvce: MessagingDataService) {}

  ngOnInit(): void {
    this.senderMessages = this.messagingSvce.getSenderMessages();
    this.userMessages = this.messagingSvce.getUserMessages();
    this.messagingSvce.userMessagesChanged.subscribe((messages: Message[]) => {
      console.log("********** messages have changed");
      this.userMessages = messages;
    });
  }
}
```

With this setup, you can now "send" messages from the UI, and the new message
will show up in the conversation thread.

> Note: even though `EventEmitter` is currently an "observable", i.e. an
> instance of `EventEmitter` can be subscribed to, this is not guaranteed to
> stay the case as Angular continues to evolve. In a later section, we will
> learn more about Observables and you will be able to use that knowledge to
> replace our `EventEmitter`-based implementation with a more general
> implementation, if you choose to do so.
