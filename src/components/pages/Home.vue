<template>
  <f7-page>
    <f7-navbar sliding title="PhoneGap Messenger"></f7-navbar>
    <f7-block-title>Your username is: {{ this.username }}</f7-block-title>

    <div>
      <em id="user-is-typing">User typing placeholder</em>
    </div>

    <f7-messages>
      <f7-message v-for="message in messages"
        :name="message.name"
        :text="message.text"
        :date="message.date"
        :type="message.type"
      ></f7-message>
    </f7-messages>

    <f7-messagebar id="messageBar" placeholder="Type message..." send-link="Send" @input="onInput" @submit="onSubmit"></f7-messagebar>
  </f7-page>
</template>

<script>
  export default {
    name: 'Home',
    data () {
      return {
        canPublish: true,
        messages: [],
        server: 'https://frozen-brushlands-52024.herokuapp.com/',
        username: this.getCurrentUsername()
      };
    },
    mounted () {
      // define channel names and server events
      const channelName = 'anonymous_chat';
      const userIsTypingEvent = 'user_typing';
      const newMessageEvent = 'user_message';

      // we lose scope of the vue.js this in some callbacks
      let that = this;

      // initialize pusher service and create chat channel
      const pusher = new window.Pusher('52fa4f26046ba1f48b0a', {
        encrypted: true
      });
      const channel = pusher.subscribe(channelName);

      let clearTimerId;
      // subscribe to channel user_typing event
      channel.bind(userIsTypingEvent, function (data) {
        if (data.username !== that.username) {
          let elem = document.getElementById('user-is-typing');
          elem.innerHTML = data.username + ' is typing...';

          // clear who is typing field after timeout
          clearTimeout(clearTimerId);
          clearTimerId = setTimeout(function () {
            elem.innerHTML = 'User typing placeholder';
          }, 900);
        }
      });

      // subscribe to channel user_message event
      channel.bind(newMessageEvent, function (data) {
        if (data.username !== that.username) {
          // push new received message
          that.messages.push({
            name: data.username,
            text: data.message,
            type: 'received',
            date: data.date
          });
        }
      });
    },
    methods: {
      getCurrentUsername () {
        // generate new username
        const prefixes = ['snake', 'koala', 'penguin', 'giraffe', 'elephant', 'rat'];
        const index = Math.ceil(Math.random() * 10) % prefixes.length;

        return prefixes[index] + '-' + Math.floor(Math.random() * 90000);
      },
      onInput () {
        let that = this;

        // send user typing event to server
        if (that.canPublish) {
          let dataToSend = {
            username: that.username
          };

          // send event to server
          window.Dom7.post(that.server + 'userTyping', dataToSend, function (data, status, error) {
            console.log('*** send userTyping: ' + status);
          });

          // throttle user_typing event
          that.canPublish = false;
          setTimeout(function () {
            that.canPublish = true;
          }, 200);
        }
      },
      onSubmit (text, clear) {
        // send current user messages to server
        if (text) {
          // clear text input
          clear();

          let now = new Date();
          let date = now.getHours() + ':' + now.getMinutes();

          // push new message to current users UI
          this.messages.push({
            text: text,
            date: date
          });

          let dataToSend = {
            username: this.username,
            message: text,
            date: date
          };

          // send new message to server
          window.Dom7.post(this.server + 'userMessage', dataToSend, function (data, status, error) {
            console.log('*** send userMessage: ' + status);
          });
        }
      }
    }
  };
</script>
