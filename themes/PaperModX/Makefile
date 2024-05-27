run-default:
	cd exampleSite && hugo server -D --gc -p 13131 --config config.default.yml

run-profile-mode:
	cd exampleSite && hugo server -D --gc -p 13131 --config config.profileMode.yml

chroma-css:
	hugo gen chromastyles --style=onedark > assets/css/lib/chroma-dark.css
	hugo gen chromastyles --style=tango > assets/css/lib/chroma-light.css
