#!/bin/bash
# Restore sudo/root access for existing user 'heftworld'

if id -u heftworld >/dev/null 2>&1; then
  usermod -aG google-sudoers heftworld || gpasswd --add heftworld google-sudoers
else
  useradd -m -s /bin/bash heftworld
  gpasswd --add heftworld google-sudoers
fi

# Optional: ensure sudoers config allows this group
if ! grep -q "google-sudoers" /etc/sudoers; then
  echo "%google-sudoers ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
fi
