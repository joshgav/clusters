## 1 - kind

Created a kind cluster with a worker node. Wouldn't boot, had to increase max_user_watches and max_user_instances for fs.inotify.

```bash
# Check current limits
cat /proc/sys/fs/inotify/max_user_watches
cat /proc/sys/fs/inotify/max_user_instances

# Temporarily increase limits (until reboot)
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512

## for permanence:

# Add to sysctl configuration
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf

# Apply the changes
sudo sysctl -p
```