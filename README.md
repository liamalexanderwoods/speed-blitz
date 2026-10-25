# Speed Blitz

A 3D superhero game that runs in the browser. You're a speedster in Blitz City, a town full of people and cars. Run at Mach 3, freeze time, chain lightning-fast blitzes through crowds, punch, slam the ground, and pick up and throw cars. Cause too much chaos and the police come after you.

## Play

Open `index.html` in a browser. There's no build step. Three.js loads from a CDN, so you need an internet connection. It works on desktop (mouse and keyboard) and on phones (touch controls).

## Controls

| Key | Action |
| --- | --- |
| WASD | Move |
| Mouse | Look |
| Shift | Super speed (up to Mach 3) |
| Space | Jump (press again to double jump, Shift+Space to super jump) |
| Left click | Super punch (hold to keep punching) |
| Right click / Q | Speed blitz |
| F | Freeze time |
| E | Ground slam |
| G | Grab / throw |
| P | FPS meter |
| H | Hide help |

On phones, use the on-screen joystick and buttons.

## Graphics quality

Click the graphics button on the title screen to switch between low, medium and high. You can also add `?q=low` to the URL.

## Test modes

Add these to the URL to run scripted checks. Results are written to the page title.

- `?test`: a scripted hero that uses every power
- `?test&still`: stand still with a high wanted level and watch the police arrive
- `?test&mach`: run into the city at full speed
- `?test&boom`: car explosion chain reaction
- `?test&bench`: performance benchmark

## Deployment

The game is served as a static asset by a Cloudflare Worker (`wrangler.jsonc`). Pushing to `main` deploys it.
