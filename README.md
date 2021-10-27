#donationalert

A basic Hive donation alert overlay system for streamers.
Customise your settings, generate a link to import into your OBS scene as a browser source.

    - Customisable alert image, sound, title, animation
    - Set donation amount minimums to trigger alerts, memo display and reading donation with text to speech 
    - Ignore specific accounts

Known issues:
    - Multiple transfers in a short timeframe will all play at once (needs a queuing system)
    - Accounts with no recent transfers in history will bug out (send a dust tranfer to fix this)

Future features that would be nice to have (pull requests welcome):
    - Support youtube video alerts, with audio
    - Tiered donations with different graphics/sound/text

Find it live at https://donationalert.ausbit.dev