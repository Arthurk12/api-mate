chokidar = require('chokidar')
crypto = require('crypto')
fs = require('fs')
{spawn} = require('child_process')
pug = require('pug')
sass = require('node-sass')

binPath = './node_modules/.bin/'

# Returns a string with the current time to print out.
timeNow = ->
  today = new Date()
  today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds()

# Spawns an application with `options` and calls `onExit`
# when it finishes.
run = (bin, options, onExit) ->
  bin = binPath + bin
  console.log timeNow() + ' - running: ' + bin + ' ' + (if options? then options.join(' ') else '')
  cmd = spawn bin, options
  cmd.stdout.on 'data', (data) -> #console.log data.toString()
  cmd.stderr.on 'data', (data) -> console.log data.toString()
  cmd.on 'exit', (code) ->
    console.log timeNow() + ' - done.'
    onExit?(code, options)

compileView = (done) ->
  options = ['--pretty', 'src/views/api_mate.pug', '--out', 'lib', '--obj', 'src/pug_options.json']
  run 'pug', options, ->
    options = ['--pretty', 'src/views/redis_events.pug', '--out', 'lib', '--obj', 'src/pug_options.json']
    run 'pug', options, ->
      done?()

compileCss = (done) ->
  options = ['src/css/api_mate.scss', 'lib/api_mate.css']
  run 'node-sass', options, ->
    options = ['src/css/redis_events.scss', 'lib/redis_events.css']
    run 'node-sass', options, ->
      done?()

compileJs = (done) ->
  options = [
    '-o', 'lib',
    '--join', 'api_mate.js',
    '--compile', 'src/js/application.coffee', 'src/js/templates.coffee', 'src/js/api_mate.coffee'
  ]
  run 'coffee', options, ->
    options = [
      '-o', 'lib',
      '--join', 'redis_events.js',
      '--compile', 'src/js/application.coffee', 'src/js/redis_events.coffee'
    ]
    run 'coffee', options, ->
      done?()

# Writes the SHA-256 of each domain in the env var API_MATE_PRODUCTION_DOMAINS
# (separated by commas or whitespace) to be matched against the server in use.
# Only the hashes are published, so the page does not give away the list, which is
# why the wildcards are limited to what the page can enumerate and hash itself:
# * `#` is any number. The page replaces all numbers in the server with `#`, so an
#   entry with `#` has all its numbers replaced too.
# * `*text*` is any server that contains `text`. The page hashes every substring of
#   the server wrapped in asterisks, which keeps it apart from a domain `text`.
normalizeProductionDomain = (domain) ->
  return domain if /^\*[^*]+\*$/.test(domain)
  domain = domain.replace(/^\*?\./, '').replace(/\.$/, '')
  if domain.indexOf('#') >= 0 then domain.replace(/\d+/g, '#') else domain

compileProductionDomains = (done) ->
  domains = (process.env.API_MATE_PRODUCTION_DOMAINS or '').toLowerCase().split(/[\s,]+/)
  domains = (normalizeProductionDomain(domain) for domain in domains)
  hashes = (crypto.createHash('sha256').update(domain).digest('hex') for domain in domains when domain)
  js = "window.apiMateProductionDomainHashes = #{JSON.stringify(hashes)};\n"
  fs.writeFileSync('lib/production_domains.js', js)
  console.log timeNow() + " - production domains: #{hashes.length}"
  done?()

build = (done) ->
  compileView (err) ->
    compileCss (err) ->
      compileJs (err) ->
        compileProductionDomains (err) ->
          done?()

watch = () ->
  watcher = chokidar.watch('src', { ignored: /[\/\\]\./, persistent: true })
  watcher.on 'all', (event, path) ->
    console.log timeNow() + ' = detected', event, 'on', path
    if path.match(/\.coffee$/)
      compileJs()
    else if path.match(/\.scss/)
      compileCss()
    else if path.match(/\.pug/)
      compileView()

task 'build', 'Build everything from src/ into lib/', ->
  build()

task 'watch', 'Watch for changes to compile the sources', ->
  watch()
