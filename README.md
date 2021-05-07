# Discord Music Bot

## Steps:
Run the following to install the dependencies (note that you need to install ffmpeg-static too)

```bash
npm install discord.js ffmpeg fluent-ffmpeg @discordjs/opus ytdl-core ffmpeg-static --save
```

-------------------------

Import `ytdl-core`
```js
const ytdl = require('ytdl-core');
```

-------------------------

Create a [`Map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) object, called `queue` so that we can store the users' songs, across ANY server the bot is in
```js
const queue = new Map();
```

-------------------------

After that, insert the following code into the `client.on("message", ...)` function
```js 
// serverQueue is an object that holds info about songs in a server
const serverQueue = queue.get(message.guild.id);

if (message.content.startsWith(`${prefix}play`)) {
    execute(message, serverQueue);
    return;
} else if (message.content.startsWith(`${prefix}skip`)) {
    skip(message, serverQueue);
    return;
} else if (message.content.startsWith(`${prefix}stop`)) {
    stop(message, serverQueue);
    return;
} else {
    message.channel.send("You need to enter a valid command!");
}
```
**Note:** put this code before the `const commandBody = ...` part

This helps the bot process the users' inputs, and then runs functions accordingly.

Notice that it calls 3 functions, namely `execute()`, `skip()`, and `stop()`

-------------------------

Now, let's define the function `execute()`
```js
async function execute(message, serverQueue) {
  const args = message.content.split(" ");

  const voiceChannel = message.member.voice.channel;
  if (!voiceChannel)
    return message.channel.send(
      "You need to be in a voice channel to play music!"
    );
  const permissions = voiceChannel.permissionsFor(message.client.user);
  if (!permissions.has("CONNECT") || !permissions.has("SPEAK")) {
    return message.channel.send(
      "I need the permissions to join and speak in your voice channel!"
    );
  }

	// everything is ok, can do stuff here...

}
```

As you can see, for now it just sets up things for us to use later (ie. get the user's voice channel and get the server's permission)

-------------------------

Next, still inside the `execute()` function, we get information about the song by using `ytdl-core`
```js
const songInfo = await ytdl.getInfo(args[1]);
const song = {
    title: songInfo.videoDetails.title,
    url: songInfo.videoDetails.video_url,
};
```

**Note:** The [`ytdl-core` library](https://www.npmjs.com/package/ytdl-core) only supports playing YouTube links

-------------------------

After fetching info about the song, we now need to add it to our queue.

Remember the `queue` Map object we created earlier?


We first check if this server now already has a queue, and if it does, we add the song to the queue using `.push()`

```js
if (!serverQueue) {

	// server has no music queue
	// create a queue for the server
	// join the channel and play music

} else {
 serverQueue.songs.push(song);
 console.log(serverQueue.songs);
 return message.channel.send(`${song.title} has been added to the queue!`);
}
```

-------------------------

Now, add the following code into the `if(!serverQueue){ ... }` block

```js
// Creating the contract for our queue
const queueContruct = {
	textChannel: message.channel,
	voiceChannel: voiceChannel,
	connection: null,
	songs: [],
	volume: 5,
	playing: true,
};
// Setting the queue using our contract
queue.set(message.guild.id, queueContruct);
// Pushing the song to our songs array
queueContruct.songs.push(song);

try {
	// Here we try to join the voicechat and save our connection into our object.
	var connection = await voiceChannel.join();
	queueContruct.connection = connection;
	// Calling the play function to start a song
	play(message.guild, queueContruct.songs[0]);
} catch (err) {
	// Printing the error message if the bot fails to join the voicechat
	console.log(err);
	queue.delete(message.guild.id);
	return message.channel.send(err);
}
```

Yes, it's very long - let's break it down:

Inside the `queue` Map() object, an example will something look like

```js
Map {
  0123456789: {
    textChannel: TextChannel {
			...
    },
    voiceChannel: VoiceChannel {
			...
    },
    connection: ... ,
    songs: [ [Object] ],
    volume: 5,
    playing: true
  }
}
```
where `0123456789` is our server (or guild) id

So, when the server has no queue, we will create one, and then save it in our `queue` object.

Then, inside `try{ ... }`, we play the music in the voice channel by calling the `play()` function.

And, we're done! This was the `execute()` function.


-------------------------

Now, create a new `play()` function
```js 

function play(guild, song) {
  const serverQueue = queue.get(guild.id);
  if (!song) {
    serverQueue.voiceChannel.leave();
    queue.delete(guild.id);
    return;
  }

	// got song, do stuff...

}

```

The code above checks if there are still any songs to play - if there is none, it will leave.

-------------------------

Still inside the `play()` function, add the following code:
```js 
const dispatcher = serverQueue.connection
    .play(ytdl(song.url))
    .on("finish", () => {
        serverQueue.songs.shift();
        play(guild, serverQueue.songs[0]);
    })
    .on("error", error => console.error(error));
dispatcher.setVolumeLogarithmic(serverQueue.volume / 5);
serverQueue.textChannel.send(`Start playing: **${song.title}**`);

```

What it does is play the song, and when the song is finished plays the next song.

**Note:** did you realise that we called the `play()` function inside the `play()` function? This is called recursion, and calls itself over and over again until something happens. (in our case plays songs until there are none left to play)

Now we're done with the `play()` function!

Try using the bot now - it should join and play music.

-------------------------

Next let's create the `skip()` function
```js 
function skip(message, serverQueue) {
  if (!message.member.voice.channel)
    return message.channel.send(
      "You have to be in a voice channel to stop the music!"
    );
  if (!serverQueue)
    return message.channel.send("There is no song that I could skip!");
  serverQueue.connection.dispatcher.end();
}
```

This is relatively simple, and stops playing the current song.

-------------------------

Moving on, we will create the `stop()` function
```js 
function stop(message, serverQueue) {
  if (!message.member.voice.channel)
    return message.channel.send(
      "You have to be in a voice channel to stop the music!"
    );
  
  if (!serverQueue)
    return message.channel.send("There is no song that I could stop!");
    
  serverQueue.songs = [];
  serverQueue.connection.dispatcher.end();
}
```

This is literally the same code as `skip()`, but we clear the `serverQueue.songs` array too.

-------------------------

Now you're done! That's the [whole discord music bot source code](https://github.com/Fogeinator/Discord-Music-Bot-Completed).


-------------------------



[Reference Article](https://gabrieltanner.org/blog/dicord-music-bot)
