FROM jbonjean/base-nodejs-8 as builder

RUN apt-get update && \
	eatmydata apt-get install -y --no-install-recommends \
		build-essential \
		libavahi-compat-libdnssd-dev \
		libudev-dev \
		python \
	&& cleanup



# Build Volumio Core.
FROM builder as core-builder

ENV CORE_VERSION=a4e73ab3bab9527bb81808272661ed8bdf4b5586

RUN mkdir -p /dist/volumio && \
	curl -L https://github.com/volumio/Volumio2/archive/$CORE_VERSION.tar.gz | \
		tar -C /dist/volumio --strip-components=1 -xz

# Remove fs-extra package that seems to cause issue on runtime. TBD.
RUN sed -i '/"fs-extra"/d' /dist/volumio/package.json

RUN cd /dist/volumio && \
	eatmydata npm install && \
	eatmydata node /dist/volumio/utils/misc/clean-node-modules.js /dist/volumio



# Build Volumio UI.
FROM builder as ui-builder

ENV UI_VERSION=7ee27489248f389cf8b3f3421d9f9efd04a720bd

RUN mkdir -p /tmp/volumio-ui && \
	curl -L https://github.com/volumio/Volumio2-UI/archive/$UI_VERSION.tar.gz | \
		tar -C /tmp/volumio-ui --strip-components=1 -xz

RUN cd /tmp/volumio-ui && \
	eatmydata npm install && \
	eatmydata bower install --allow-root && \
	eatmydata gulp build --theme="volumio" --env="production" && \
	mkdir -p /dist && mv /tmp/volumio-ui/dist /dist/volumio



# Build final image.
FROM jbonjean/base-nodejs-8

ARG DOCKER_TAG=latest

RUN apt-get update && \
	eatmydata apt-get install -y --no-install-recommends \
		alsa-utils \
		avahi-daemon \
		avahi-utils \
		dbus \
		libavahi-compat-libdnssd1 \
		libnss-mdns \
		minizip \
		mpc \
		mpd \
		pulseaudio \
		shairport-sync \
		sudo \
		wget \
	&& cleanup

# Add specific user and group (hardcoded).
RUN groupadd -g 1000 volumio && useradd -u 1000 -g 1000 volumio && \
	# add users to audio group
	usermod -a -G audio volumio && usermod -a -G audio mpd

# Install dist from previous stages.
COPY --from=core-builder /dist/volumio /volumio
COPY --from=ui-builder /dist/volumio /volumio/http/www

# Symlink library to /mnt for volumio.
RUN rm -rf /mnt && ln -s /var/lib/mpd/music /mnt && \
	# Create node symlink in /usr/local (hardcoded).
	ln -s /opt/node/bin/node /usr/local/bin/node

# Copy services and config.
COPY etc /etc

# A few additional hacks to finalize the image.
COPY volumio-image-hacks /tmp/
RUN /tmp/volumio-image-hacks
