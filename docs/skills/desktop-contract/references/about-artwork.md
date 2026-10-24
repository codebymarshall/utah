## About artwork and the panel icon (#318)

The About page's OS name is separate from its artwork. GNOME 51's
`setup_os_logo` uses `DISTRIBUTOR_LOGO` and `DARK_MODE_DISTRIBUTOR_LOGO` before
consulting os-release `LOGO`; Utah's spec configures Fedora's
`fedora_logo_med.png`/`fedora_whitelogo_med.png` paths, or the RHEL
`fedora-logo.png`/`system-logo-white.png` pair. Common ships Bluefin artwork
under all four names, but the logos-RPM swap erases the early overlay.
The desktop RUN bind-mounts Common's pinned pixmaps and branding restores
those four files after the RPM transaction without adding a COPY layer.
The package installer checks for residual distro artwork before this restore;
the desktop contract then checks the restored Bluefin assets.

The pinned command-menu extension creates `St.Icon` from `menuicon-setting`.
Keep the icon *name* `ublue-logo-symbolic`, and the command labels/order/
location and help URLs, unchanged. The art behind that name is artwork's
Utahraptor menu logo (`assets/vector/icons/logo-menu/logo-utahraptor-menu.svg`,
a white ring with the raptor, installed verbatim from the commit in
`config/artwork.lock` by the `artwork` build stage; the previous Bluefin
"ublue" mark is gone). The contract requires the SVG and the compiled hicolor
cache, which branding regenerates after all overlays. The menu must show the
mark (not a placeholder) and retain its commands.

Wallpapers: `/usr/share/backgrounds/bluefin/*` and
`/usr/share/gnome-background-properties/*.xml` come from the pinned
`bluefin-wallpapers-gnome` image, overlaid after Common's copy (see the
Containerfile skill). `01-bluefin.xml` and `06-bluefin.xml` are contract files.
The mascot the live installer shows is the Utah black-outline art, staged into
`iso/live/src/images/utahraptor.png` by the ISO builders; the os-release `LOGO`
raptor (`fedora-logo-icon.png`) and the About-page pixmaps remain Common's.

## GDM greeter logo

`/usr/share/pixmaps/bluefin-gdm-logo.png` is artwork's
`assets/vector/logos/bluefin/bluefin-logo-dark.svg` (white lettering for the
dark greeter) at the commit in `config/artwork.lock`, verified and staged by the
`artwork` build stage and bind mounted into the branding RUN.
`scripts/configure-branding.sh` rasterizes it after the package transaction
into a `GDM_LOGO_MAX_WIDTH`x`GDM_LOGO_MAX_HEIGHT` (150x64) box, preserving the
aspect ratio (150x61), using librsvg2's GObject typelib through `python3-gobject`
and `python3-cairo`. The image has no `rsvg-convert` and the build stage has no
rasterizer, so the render cannot move earlier; Fedora 44's librsvg2 ships the
typelib but no gdk-pixbuf loader. It is written under a Utah-owned name so no
logos RPM owns or erases it.
