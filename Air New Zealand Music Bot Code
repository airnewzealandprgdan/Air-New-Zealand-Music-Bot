const { Client, GatewayIntentBits, REST, Routes, SlashCommandBuilder } = require('discord.js');
const { joinVoiceChannel, createAudioPlayer, createAudioResource, getVoiceConnection } = require('@discordjs/voice');
const ytdl = require('@distube/ytdl-core');

const token = process.env.TOKEN;
const clientId = '1404484459977244683';

const client = new Client({ intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildVoiceStates] });

// Global variables to store player and queue
let currentPlayer = null;
let songQueue = [];
let currentConnection = null;

const commands = [
  new SlashCommandBuilder()
    .setName('play')
    .setDescription('Play a YouTube song')
    .addStringOption(opt => opt.setName('url').setDescription('YouTube URL').setRequired(true)),
  
  new SlashCommandBuilder()
    .setName('stop')
    .setDescription('Stop the music and clear the queue'),
  
  new SlashCommandBuilder()
    .setName('skip')
    .setDescription('Skip the current song'),
  
  new SlashCommandBuilder()
    .setName('queue')
    .setDescription('Show the current music queue'),
  
  new SlashCommandBuilder()
    .setName('volume')
    .setDescription('Set the volume (0-100)')
    .addIntegerOption(opt => opt.setName('level').setDescription('Volume level (0-100)').setRequired(true)),
  
  new SlashCommandBuilder()
    .setName('join')
    .setDescription('Make the bot join your voice channel')
].map(cmd => cmd.toJSON());

const rest = new REST({ version: '10' }).setToken(token);

client.once('ready', async () => {
  console.log(`Logged in as ${client.user.tag}!`);
  
  // Register commands for each guild the bot is in
  for (const guild of client.guilds.cache.values()) {
    try {
      await rest.put(Routes.applicationGuildCommands(clientId, guild.id), { body: commands });
      console.log(`Slash commands registered for guild: ${guild.name}`);
    } catch (err) { 
      console.error(`Error registering commands for guild ${guild.name}:`, err); 
    }
  }
  
  console.log('All slash commands ready!');
});

async function playNextSong(interaction) {
  if (songQueue.length === 0) {
    if (interaction) {
      await interaction.editReply('Queue is empty!');
    }
    return;
  }

  const nextSong = songQueue.shift();
  
  try {
    console.log(`🎵 Attempting to play: ${nextSong.url}`);
    
    const stream = ytdl(nextSong.url, { 
      filter: 'audioonly',
      highWaterMark: 1 << 25,
      quality: 'highestaudio'
    });
    
    const resource = createAudioResource(stream);
    
    currentPlayer.play(resource);
    
    if (interaction) {
      await interaction.editReply(`🎵 Now playing: ${nextSong.url}`);
    }
    
    // Add error handling for the player
    currentPlayer.on('error', error => {
      console.error('❌ Audio player error:', error);
    });
    
    // Add connection error handling
    currentConnection.on('error', error => {
      console.error('❌ Voice connection error:', error);
    });
    
    // When song ends, play next song
    currentPlayer.on('stateChange', (oldState, newState) => {
      console.log(`🔄 Player state changed: ${oldState.status} → ${newState.status}`);
      if (newState.status === 'idle' && songQueue.length > 0) {
        playNextSong();
      }
    });
    
  } catch (error) {
    console.error('Error playing next song:', error);
    if (interaction) {
      await interaction.editReply('❌ Error playing next song.');
    }
  }
}

client.on('interactionCreate', async interaction => {
  if (!interaction.isChatInputCommand()) return;
  
  try {
    await interaction.deferReply();
    
    switch (interaction.commandName) {
      case 'play':
        const url = interaction.options.getString('url');
        
        if (!ytdl.validateURL(url)) {
          return await interaction.editReply('Please provide a valid YouTube URL.');
        }
        
        const vc = interaction.member.voice.channel;
        if (!vc) {
          return await interaction.editReply('Join a voice channel first!');
        }
        
        // Add song to queue
        songQueue.push({ url, requester: interaction.user.username });
        
        // If no connection exists, create one
        if (!currentConnection) {
          currentConnection = joinVoiceChannel({ 
            channelId: vc.id, 
            guildId: vc.guild.id, 
            adapterCreator: vc.guild.voiceAdapterCreator 
          });
          
          currentPlayer = createAudioPlayer();
          currentConnection.subscribe(currentPlayer);
        }
        
        // If nothing is playing, start playing
        if (currentPlayer.state.status === 'idle') {
          await playNextSong(interaction);
        } else {
          await interaction.editReply(`Added to queue: ${url} (Position: ${songQueue.length})`);
        }
        break;
        
      case 'stop':
        if (!currentPlayer) {
          return await interaction.editReply('No music is playing!');
        }
        
        currentPlayer.stop();
        songQueue = [];
        await interaction.editReply('⏹️ Music stopped and queue cleared!');
        break;
        
      case 'skip':
        if (!currentPlayer || currentPlayer.state.status === 'idle') {
          return await interaction.editReply('No music is playing!');
        }
        
        // Check if user is in the same voice channel as the bot
        const userVoiceChannel = interaction.member.voice.channel;
        if (!userVoiceChannel) {
          return await interaction.editReply('❌ You need to be in a voice channel to skip songs!');
        }
        
        if (currentConnection && currentConnection.joinConfig.channelId !== userVoiceChannel.id) {
          return await interaction.editReply('❌ You need to be in the same voice channel as the bot to skip songs!');
        }
        
        currentPlayer.stop();
        
        if (songQueue.length > 0) {
          await playNextSong(interaction);
        } else {
          await interaction.editReply('⏭️ Song skipped! Queue is now empty.');
        }
        break;
        
      case 'queue':
        if (songQueue.length === 0) {
          return await interaction.editReply('Queue is empty!');
        }
        
        let queueText = '🎵 **Current Queue:**\n';
        songQueue.forEach((song, index) => {
          queueText += `${index + 1}. ${song.url} (requested by ${song.requester})\n`;
        });
        
        await interaction.editReply(queueText);
        break;
        
      case 'volume':
        const volume = interaction.options.getInteger('level');
        
        if (volume < 0 || volume > 100) {
          return await interaction.editReply('Volume must be between 0 and 100!');
        }
        
        if (!currentPlayer) {
          return await interaction.editReply('No music is playing!');
        }
        
        // Note: Discord.js voice doesn't have built-in volume control
        // This is a placeholder - you'd need additional libraries for actual volume control
        await interaction.editReply(`🔊 Volume set to ${volume}% (Note: Volume control requires additional setup)`);
        break;
        
      case 'join':
        const voiceChannel = interaction.member.voice.channel;
        if (!voiceChannel) {
          return await interaction.editReply('You need to be in a voice channel!');
        }
        
        if
