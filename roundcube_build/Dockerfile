FROM roundcube/roundcubemail:latest

# Patch the default session lifetime from 10 minutes to 525600 (1 year)
# We modify the source template so it survives resets
RUN sed -i "s/\$config\['session_lifetime'\] = 10;/\$config\['session_lifetime'\] = 525600;/g" /usr/src/roundcubemail/config/defaults.inc.php
