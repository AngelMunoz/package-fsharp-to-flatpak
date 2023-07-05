# Package an F# Avalonia app as a Flatpak

> **_Note_**: While this example is written for F#, it should work for any .NET app.

You're done with your app. Providing binaries is the next step, but you want to make it easy for your users to install and run your app. You also want to make it easy for you to distribute your app and even maybe auto-updates. You've heard about Flatpaks and want to try it out.

This project is based on this _https://docs.flatpak.org/en/latest/first-build.html_ tutorial which has better information in regards of the flatpak build infrastructure.

> **_Note_**:For this you should have set already the `org.flatpak.Builder`, `org.freedesktop.Platform`, and `org.freedesktop.Sdk` flatpaks. If you haven't, you can do so by following the instructions here: _https://docs.flatpak.org/en/latest/first-build.html#installing-the-flatpak-sdk_.

---

Before we get into the juicy parts of the flatpak stuff. Let's build the app first.

```
dotnet publish ./src -c Release -o ./flatpak/Flaco --self-contained -r linux-x64.
```

in this case we'll use a self-contained app so we don't have to depend on the flatpak dotnet sdk or runtime.

Ideally we just want to call a simple command and get running, nothing else.

## The Flatpak parts

We will need a `flatpak` directory mainly for convenience and also to be sure we have what we need to build the flatpak.

The manifest is relatively simple

```yaml
# your app's id which ideally should match the manifest file name
app-id: org.flatpak.Flaco
runtime: org.freedesktop.Platform
runtime-version: "22.08" # version of the runtime
sdk: org.freedesktop.Sdk # the sdk to use
command: Flaco # the command to run, basically the command that starts the app
finish-args:
  # Flatpaks run in sandbox mode. This is the list of permissions we need
  # As we're running an Avalonia App, we need to specify that we want access
  # to the window infrastructure of the host
  - --socket=fallback-x11
  # for more information visit https://docs.flatpak.org/en/latest/sandbox-permissions.html#sandbox-permissions

modules:
  # This is our app and the ;build' instructions for it
  - name: Flaco
    buildsystem: simple
    # In our case we've already run `dotnet publish` so we just need to copy the
    # outputs to the right place
    build-commands:
      - mkdir -p /app/bin
      - mv ./app-sources /app/bin/app-sources
      # allow our app to be executed
      - chmod +x /app/bin/app-sources/Flaco
      # create a symlink to the executable
      - ln -s /app/bin/app-sources/Flaco /app/bin/Flaco
    # This is the list of files/directories we want to copy to the flatpak
    # for more elaborated builds we could fetch a zip file, git repository or
    # even build from source however that comes with different challenges
    # so we'll leave that for another time
    sources:
      - type: dir
        # This is the path on the host machine './flatpak/Flaco'
        # but since we're running the flatpak-builder command in the flatpak directory
        # we'll omit the other one
        path: Flaco
        # This is the path inside the flatpak which gets copied to the root of the flatpak
        dest: app-sources
```

Once we've built our dotnet app and have the manifest ready, we can build the flatpak.

- `flatpak-builder --user --install --force-clean build-dir org.flatpak.Flaco.yml`
  - `--user`: we want to install the flatpak for the current user
  - `--install`: we want to install the flatpak after building it
  - `--force-clean`: we want to clean the build directory before building the flatpak
- `flatpak run org.flatpak.Flaco`

And that's it. The app should show on the screen and now we have a flatpak for our app tested and ready to be published in flathub!

### Extra Notes

Thanks to (@scitesy)[https://social.librem.one/@scitesy/110662886460321742] for providing me great resources to explore this area as well as the flatpak documentation.
Also As noted by [@nlogozzo](https://mastodon.social/@nlogozzo/110662954309000176) (so much thanks, this helped me quite a lot!) when you run `flatpak-builder` you're not allowed to access the network anymore, so if you want to build from source your dotnet code you need to make a couple of extra workaround mainly involving the `sources` section of the manifest. So the dotnet sdk and NuGet have something to work with.

As you may guess, the way we're building this right now makes apps quite heavy as we're not doing any trimming, or using any of the flatpak features to make the app lighter. But this is a good starting point.

The following point could be to build the app as a runtime dependant app which can in turn use the dotnet runtime provided by the flatpak sdk. This would make the app lighter and also would allow us to use the flatpak to be more streamlined into it.
