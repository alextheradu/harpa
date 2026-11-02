# harpa

a customizable (kinda) cozy lo-fi screen for Hack Club Haven. meant for projectors!

## running it

```sh
npm install
npm run dev
```

you'll need node 22.17+.

## making it yours

the first time you open it, it asks for your haven's logo and a soundcloud playlist. skip whatever you want. it all saves in that browser, so set it up on the laptop hooked to the projector. wiggle the mouse and a gear shows up top right if you want to change stuff later.

edits to the soundcloud playlist show up on the site within a minute or so. your keyboard's play/pause/skip keys work too.

## deploying

```sh
npm run build
rm -rf docs && cp -R build docs
```
