# i3 Configuration

## Special keys
First off all, I wanted to know the name of all my keys in order to map them to whatever usage


To do so, run `xev |grep keycode` and press your special keys.  
You might see some keyname like XF86... You can use this name in i3 for binding  
Here is my config for those keys
```bash
# Use pactl to adjust volume in PulseAudio.
set $refresh_i3status killall -SIGUSR1 i3status
bindsym XF86AudioRaiseVolume exec --no-startup-id pactl set-sink-volume @DEFAULT_SINK@ +10% && $refresh_i3status && notify-send -t 1000 -r 101 "Volume $(pactl get-sink-volume @DEFAULT_SINK@ |grep "[0-9]*%" -o |head -1)"
bindsym XF86AudioLowerVolume exec --no-startup-id pactl set-sink-volume @DEFAULT_SINK@ -10% && $refresh_i3status && notify-send -t 1000 -r 101 "Volume $(pactl get-sink-volume @DEFAULT_SINK@ |grep "[0-9]*%" -o |head -1)"
bindsym XF86AudioMute exec --no-startup-id pactl set-sink-mute @DEFAULT_SINK@ toggle && $refresh_i3status && notify-send -t 1000 -r 101 "Speakers are $(~/.config/i3/scripts/is_muted out)"
bindsym XF86AudioMicMute exec --no-startup-id pactl set-source-mute @DEFAULT_SOURCE@ toggle && $refresh_i3status && notify-send -t 1000 -r 101 "Mic is $(~/.config/i3/scripts/is_muted in)"

# audio control
bindsym XF86AudioPlay exec playerctl play && notify-send -t 1000 -r 101 "Audio play"
bindsym XF86AudioStop exec playerctl pause && notify-send -t 1000 -r 101 "Audio pause"
bindsym XF86AudioNext exec playerctl next ; notify-send -t 1000 -r 101 "test"
bindsym XF86AudioRewind exec playerctl previous && notify-send -t 1000 -r 101 "test back"

# Brightness control
bindsym XF86MonBrightnessUp exec brightnessctl set +5% && notify-send "$(brightnessctl i|grep Current)" -t 1000 -r 100
bindsym XF86MonBrightnessDown exec brightnessctl set 5%- && notify-send "$(brightnessctl i|grep Current)" -t 1000 -r 100

# Screenshots
bindsym Print exec scrot ~/Pictures/screenshots/%Y-%m-%d-%T-screen.png && notify-send "Saved in as $(date +"%Y-%m-%d-%T")-screen.png"

# Keyboard backlight
bindsym XF86KbdBrightnessUp exec ~/.config/i3/scripts/keyboard_backlight up
bindsym XF86KbdBrightnessDown exec ~/.config/i3/scripts/keyboard_backlight down

# Asus specific
bindsym XF86Launch1 exec xrandr --auto
bindsym XF86Launch4 exec ~/.config/i3/scripts/asus_get_profile toggle
```

## Drop down terminal
I wanted a terminal that I can pop in every workspace. To do so, I used the i3 `scratchpad` functionality

```bash
# Dropdown normal terminal
bindsym $mod+u [class="Xfce4-terminal" window_role="dropdown"] scratchpad show; [class="Xfce4-terminal" window_role="dropdown"] move position center
for_window [class="Xfce4-terminal" window_role="dropdown"] floating enable
for_window [class="Xfce4-terminal" window_role="dropdown"] move scratchpad
exec --no-startup-id xfce4-terminal --role=dropdown

# Dropdown python term
bindsym $mod+j [class="Xfce4-terminal" window_role="python"] scratchpad show; [class="Xfce4-terminal" window_role="python"] move position center
for_window [class="Xfce4-terminal" window_role="python"] floating enable
for_window [class="Xfce4-terminal" window_role="python"] move scratchpad
exec --no-startup-id xfce4-terminal --role=python -e 'python3'
```

## Named workspaces
All my workspaces takes the name of the app running in it.  
If this app is known, an icon is used, else the name is displayed.  

I used [this repo](https://github.com/cboddy/i3-workspace-names-daemon).  
You just have to install it `sudo pip3 install i3-workspace-names-daemon`.  
And start it in i3 config `exec_always --no-startup-id exec i3-workspace-names-daemon`

## i3 bar
I'm using **Polybar** for my status bar. It has a lot of internal modules for battery, network, temp, ... and you can easily create new ones.  
Here is the [documentation](https://github.com/polybar/polybar/wiki)

And you can find my config [here](res/polybar_config)