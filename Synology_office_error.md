Date: 2017-11-05
Title: Synology office error: User home service is not enabled
Tags:  Synology
Toc:no
Status: public
Position: 1

I'v got an Synology with DSM 6.0,and I wish to try Synology Office.But after installed office and launch,it gives an error message:User home service is not enabled.Then I go to the control panel->users->user home ,click "enable user home service" and apply and relaunch office,it still gives me the error.
I'v tried many solutions,finally got it.If you had the same problem with "user home service",just checkout your "user home folder"through terminal.

- login  through terminal
- execute "ls -lah /var/services" and see if there's a folder named "homes",and it must be symbolic linked to "/volume1/homes".
- if it's a real folder(just as my problem),then delete/backup it ,and "ln -s /volume1/homes /var/services/"
- now everything should worked fine





