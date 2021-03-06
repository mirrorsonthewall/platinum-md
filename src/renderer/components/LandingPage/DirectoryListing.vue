<template>
  <div>
    <b-container class="toolbar py-2 m-0">
      <b-row align-v="center">
        <b-col>
          <b>{{ selected.length }}</b> tracks selected {{ conversionMode }}<br />
          <b-spinner small varient="success" label="Small Spinner" v-if="progress != 'Idle'"></b-spinner> <span v-if="progress"><b-badge class="text-uppercase"><span v-if="progress != 'Idle'">{{ processing }} - {{ selected.length }} / </span>Status: {{ progress }}</b-badge></span>
        </b-col>
        <b-col class="text-right">
          <b-button variant="primary" @click="chooseDir">Folder <font-awesome-icon icon="folder-open"></font-awesome-icon></b-button>
          <b-button variant="outline-light" @click="readDirectory">Rescan <font-awesome-icon icon="sync-alt"></font-awesome-icon></b-button>
          <b-button variant="success" @click="upload">Transfer <font-awesome-icon icon="angle-double-right"></font-awesome-icon></b-button>
        </b-col>
      </b-row>
    </b-container>
    
    <b-table
      selectable
      striped
      select-mode="range"
      selectedVariant="success"
      :items="files"
      :fields="fields"
      @row-selected="rowSelected"
      responsive="sm"
      :busy="isBusy"
      sortable="true"
      sortBy="title"
    >
      <div slot="table-busy" class="text-center text-danger my-2">
        <b-spinner class="align-middle"></b-spinner>
        <strong>Loading...</strong>
      </div>
      
      <template v-slot:cell(bitrate)="data">
        <div class="text-right">
          <b-badge variant="success" class="text-uppercase">{{ data.item.bitrate }} {{ data.item.codec }}</b-badge>
        </div>
      </template>
      
    </b-table>
 
  </div>
</template>

<script>
import bus from '@/bus'
import { atracdencPath, ffmpegPath, netmdcliPath } from '@/binaries'
const fs = require('fs-extra')
const readChunk = require('read-chunk')
const fileType = require('file-type')
const mm = require('music-metadata')
const { remote } = require('electron')
const Store = require('electron-store')
const store = new Store()
export default {
  data () {
    return {
      dir: '',
      files: [],
      file: '',
      progress: 'Idle',
      isBusy: true,
      fields: [
        { key: 'title', sortable: true },
        { key: 'artist', sortable: true },
        { key: 'album', sortable: true },
        { key: 'trackNo', sortable: true },
        'bitrate'
      ],
      selected: [],
      processing: 0,
      config: {},
      conversionMode: 'SP',
      bitrate: 128
    }
  },
  created () {
    this.readConfig()
    bus.$on('config-update', () => {
      this.readConfig()
    })
  },
  mounted () {
    this.readDirectory()
  },
  methods: {
    chooseDir: function () {
      remote.dialog.showOpenDialog({
        properties: ['openDirectory'],
        defaultPath: this.dir
      }, names => {
        if (names != null) {
          console.log('selected directory:' + names[0])
          store.set('baseDirectory', names[0] + '/')
          this.dir = store.get('baseDirectory')
          this.readDirectory()
        }
      })
    },
    /**
      * Reads the selected directory and displays all compatable files
      * At this stage it's simply MP3's that contain metadata
      * TODO: this can support direct WAV and FLAC input quite easily...
      * TODO: there should be more promises used here
      */
    readDirectory: function () {
      this.isBusy = true
      this.files = []
      // read the target directory
      console.log(this.dir)
      fs.readdir(this.dir, (err, dir) => {
        // loop through results
        for (let filePath of dir) {
          // ensure that we're only working with files
          if (fs.statSync(this.dir + filePath).isFile()) {
            let buffer = readChunk.sync(this.dir + filePath, 0, fileType.minimumBytes)
            let fileTypeInfo = fileType(buffer)
            // only interestedin MP3 files at the moment, ignore all others
            if (fileTypeInfo != null) {
              if (fileTypeInfo.ext === 'mp3' || fileTypeInfo.ext === 'wav' || fileTypeInfo.ext === 'flac') {
                // read metadata
                mm.parseFile(this.dir + filePath, {native: true})
                  .then(metadata => {
                    console.log(metadata)
                    // Get data for file object
                    let artist = metadata.common.artist
                    let title = metadata.common.title
                    let album = metadata.common.album
                    let bitrate = metadata.format.bitrate
                    let codec = metadata.format.codec
                    let trackNo = metadata.common.track.no
                    // write the relevent data to the files array
                    this.files.push({
                      fileName: filePath,
                      artist: (artist !== null) ? artist.substring(0, 50).replace('/', '_') : '',
                      title: (title !== null) ? title.substring(0, 50).replace('/', '_') : '',
                      album: (album !== null) ? album : '',
                      trackNo: (trackNo !== null) ? trackNo : 0,
                      format: fileTypeInfo.ext,
                      bitrate: (bitrate !== null) ? bitrate / 1000 + 'kbps' : '-',
                      codec: (codec !== null) ? codec : ''
                    })
                  })
                  .catch(err => {
                    console.error(err.message)
                  })
              }
            }
          }
        }
        // this.files = dir
        if (err) { console.log(err) }
      })
      this.isBusy = false
    },
    /**
      * Track which rows are selected
      */
    rowSelected: function (items) {
      this.selected = items
    },
    /**
      * Upload (and convert) selected tracks to player
      * This is async so it happens in order and awaits each upload to finish
      */
    upload: async function () {
      // loop through each selected track one-by-one
      for (var i = 0, len = this.selected.length; i < len; i++) {
        // create a temp directory for working files
        // TODO: maybe some automated cleanup?
        this.processing = i + 1
        var fileName = this.selected[i].fileName
        let tempDir = 'pmd-temp/'
        try {
          this.ensureDirSync(this.dir + 'pmd-temp')
          console.log('Directory created')
        } catch (err) {
          console.error(err)
        }
        var filePath = this.dir + fileName
        console.log(filePath)
        var sourceFile = filePath
        let fileExtension = '.' + sourceFile.split('.').pop()
        var destFile = this.dir + tempDir + fileName.replace(fileExtension, '.raw.wav')
        var atracFile = this.dir + tempDir + fileName.replace(fileExtension, '.at3')
        var finalFile = this.dir + tempDir + this.selected[i].title + ' - ' + this.selected[i].artist + '.wav' // filepath.replace('.mp3', '.wav')
        let self = this
        // If sending in SP mode
        // Convert to Wav and send to NetMD Device
        // The encoding process is handled by the NetMD device
        console.log('Starting conversion in <' + this.conversionMode + '> mode')
        if (this.conversionMode === 'SP') {
          await self.convertToWav(sourceFile, finalFile)
            .then(await function () {
              return self.sendToPlayer(finalFile)
            })
        // uploading as LP2
        // This uses an experimental ATRAC3 encoder
        // The files are converted into ATRAC locally, and then sent to the NetMD device
        } else {
          await self.convertToWav(sourceFile, destFile, fileExtension)
            .then(await function () {
              return self.convertToAtrac(destFile, atracFile)
            })
            .then(await function () {
              return self.convertToWavWrapper(atracFile, finalFile)
            })
            .then(await function () {
              return self.sendToPlayer(finalFile)
            })
        }
      }
    },
    /**
      * Convert input MP3 to WAV file using ffmpeg
      * This MUST be 44100 and 16bit for the atrac encoder to work
      */
    convertToWav: function (source, dest, conversionMode) {
      this.progress = 'Converting to Wav'
      return new Promise((resolve, reject) => {
        // check the filetype, and choose the output
        let convertTo = 'pcm_u8'
        if (this.conversionMode === 'SP') {
          convertTo = 'pcm_s16le'
        }
        // spawn this task and resolve promise on close
        console.log('Starting WAV conversion process using ffmpeg: ' + source + ' --> ' + dest)
        let ffmpeg = require('child_process').spawn(ffmpegPath, ['-y', '-i', '"' + source + '"', '-acodec', convertTo, '-ar', '44100', '"' + dest + '"'], { shell: true })
        ffmpeg.on('close', (code) => {
          console.log('ffmpeg returned code ' + code)
          if (code === 0) {
            resolve()
          } else {
            console.log(ffmpeg)
            reject(code)
          }
        })
        ffmpeg.on('error', (error) => {
          console.log(`ffmpeg errored with error ${error}`)
          reject(error)
        })
        ffmpeg.stdout.on('data', data => {
          // get response from ffmpeg
          console.log(data.toString())
        })
      })
    },
    /**
      * Pass WAV file to atracdenc
      * TODO: this is the longest part of the process, some user progress % feedback would be nice...
      */
    convertToAtrac: function (source, dest) {
      this.progress = 'Converting to Atrac'
      return new Promise((resolve, reject) => {
        if (fs.existsSync(dest)) {
          resolve()
        }
        // spawn this task and resolve promise on close
        let atracdenc = require('child_process').spawn(atracdencPath, ['-e', 'atrac3', '-i', source, '-o', dest, '--bitrate', this.bitrate])
        console.log(atracdenc)
        atracdenc.on('close', (code) => {
          if (code === 0) {
            console.log('atracdenc returned Success code ' + code)
            resolve()
          }
        })
        atracdenc.on('error', (error) => {
          console.log(`atracdenc errored with error ${error}`)
          reject(error)
        })
        // we get a fair bit of useful progress data returned here for debugging
        atracdenc.stdout.on('data', data => {
          console.log(data.toString())
        })
      })
    },
    /**
      * Apply the WAV wrapper around the converted file
      * This is required to be able to send using netmdcli
      */
    convertToWavWrapper: function (source, dest) {
      this.progress = 'Adding Wav Wrapper'
      return new Promise((resolve, reject) => {
        let ffmpeg2 = require('child_process').spawn(ffmpegPath, ['-y', '-i', source, '-c', 'copy', dest])
        // console.log(ffmpeg2)
        ffmpeg2.on('close', (code) => {
          console.log(`child process exited with code ${code}`)
          resolve()
        })
        ffmpeg2.on('error', (error) => {
          console.log(`child process creating error with error ${error}`)
          reject(error)
        })
      })
    },
    /**
      * Send final file using netmdcli
      * The filename is named {title} - {artist}
      */
    sendToPlayer: async function (file) {
      this.progress = 'Sending to Player'
      console.log('Attempting to send to NetMD device')
      // send off command, we wrap this so it can be retryed
      // not 100% on this method, may refactor in the future
      let retries = 5
      for (let i = 0, len = retries; i < len; i++) {
        try {
          await this.sendCommand(file)
          break
        } catch (err) {
          console.log('Attempt to send file failed, retrying...')
          await new Promise((resolve, reject) => setTimeout(resolve, 2000))
        }
      }
    },
    sendCommand: function (file) {
      return new Promise((resolve, reject) => {
        let netmdcli = require('child_process').spawn(netmdcliPath, ['-v', 'send', file])
        netmdcli.on('close', (code) => {
          if (code === 0) {
            console.log('netmdcli send returned Success code ' + code)
            bus.$emit('track-sent')
            resolve()
          } else {
            console.log('netmdcli error, returned ' + code)
            reject(code)
          }
          this.progress = 'Idle'
        })
        netmdcli.on('error', (error) => {
          console.log(`netmdcli send errored with error ${error}`)
          reject(error)
        })
        // we get a fair bit of useful progress data returned here for debugging
        netmdcli.stdout.on('data', data => {
          console.log(data.toString())
        })
      })
    },
    /**
      * Make sure temp directory actually exists, if not create it
      */
    ensureDirSync: function (dirpath) {
      try {
        fs.mkdirSync(dirpath, { recursive: true })
      } catch (err) {
        if (err.code !== 'EEXIST') throw err
      }
    },
    /**
      * Read-in config file
      */
    readConfig: function () {
      if (store.has('baseDirectory')) {
        this.dir = store.get('baseDirectory')
      }
      if (store.has('conversionMode')) {
        this.conversionMode = store.get('conversionMode')
      }
    }
  }
}
</script>
