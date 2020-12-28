
FROM danielschulz/alpine:v3.12
MAINTAINER Daniel Schulz <danielschulz2005@hotmail.com>
LABEL BUILD_TIME_PYTHON="20201228_1455 UTC"

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:${PATH}

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# runtime dependencies
RUN set -eux; \
	apk add --no-cache \
# install ca-certificates so that HTTPS works consistently
		ca-certificates \
# and tzdata for PEP 615 (https://www.python.org/dev/peps/pep-0615/)
		tzdata \
	;
# other runtime dependencies for Python are installed later

# use 3.6, 3.7, 3.8, 3.9, (as of now, use "master" for 3.10)
ARG CPYTHON_GIT_URI="https://github.com/danielschulz/cpython.git"
ARG CPYTHON_GIT_REFERENCE="v3.6.256rc"
ARG PYTHON_VERSION=${CPYTHON_GIT_REFERENCE}

ENV PYTHON_VERSION=${PYTHON_VERSION}

RUN set -ex \
	&& apk add --no-cache --virtual .fetch-deps \
		gnupg \
		tar \
		xz \
		git \
	\
	&& mkdir -p /usr/src/python \
	&& time git clone \
	     -b ${CPYTHON_GIT_REFERENCE} \
	     --depth 1 \
	     --single-branch \
	     --recurse-submodules ${CPYTHON_GIT_URI} \
	     /usr/src/python \
	&& rm -rf /usr/src/python/.git \
	&& apk add --no-cache --virtual .build-deps  \
		bluez-dev \
		bzip2-dev \
		coreutils \
		dpkg-dev dpkg \
		expat-dev \
		findutils \
		gcc \
		gdbm-dev \
		libc-dev \
		libffi-dev \
		libnsl-dev \
		libtirpc-dev \
		linux-headers \
		make \
		ncurses-dev \
		openssl-dev \
		pax-utils \
		readline-dev \
		sqlite-dev \
		tcl-dev \
		tk \
		tk-dev \
		util-linux-dev \
		xz-dev \
		zlib-dev \
# add build deps before removing fetch deps in case there's overlap
	&& apk del --no-network .fetch-deps \
	\
	&& cd /usr/src/python \
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& ./configure \
		--build="${gnuArch}" \
		--enable-loadable-sqlite-extensions \
		--enable-optimizations \
		--enable-option-checking=fatal \
		--enable-shared \
		--with-system-expat \
		--with-system-ffi \
		--without-ensurepip \
	&& make -j "$( nproc --ignore=1 )" \
# set thread stack size to 1MB so we don't segfault before we hit sys.getrecursionlimit()
# https://github.com/alpinelinux/aports/commit/2026e1259422d4e0cf92391ca2d3844356c649d0
		EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000" \
		LDFLAGS="-Wl,--strip-all" \
	&& make install \
	&& rm -rf /usr/src/python \
	\
	&& find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name '*.a' \) \) \
		\) -exec rm -rf '{}' + \
	\
	&& find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec scanelf --needed --nobanner --format '%n#p' '{}' ';' \
		| tr ',' '\n' \
		| sort -u \
		| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
		| xargs -rt apk add --no-cache --virtual .python-rundeps \
	&& apk del --no-network .build-deps \
	\
	&& python3 --version

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
	&& ln -s idle3 idle \
	&& ln -s pydoc3 pydoc \
	&& ln -s python3 python \
	&& ln -s python3-config python-config

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 20.3.3
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL https://raw.githubusercontent.com/pypa/get-pip/${PYTHON_PIP_VERSION}/get-pip.py
ENV PYTHON_GET_PIP_SHA256 6a0b13826862f33c13b614a921d36253bfa1ae779c5fbf569876f3585057e9d2

RUN set -ex; \
	\
	wget -q -O get-pip.py "${PYTHON_GET_PIP_URL}"; \
	echo "${PYTHON_GET_PIP_SHA256} *get-pip.py" | sha256sum -c -; \
	\
	python get-pip.py \
		--disable-pip-version-check \
		--no-cache-dir \
		"pip==${PYTHON_PIP_VERSION}" \
	; \
	pip --version; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' +; \
	rm -f get-pip.py

RUN echo -e "Current Python version is $( python -V ) \n(found in $( which python ))." && \
    echo -e "Current PIP version is $( pip -V ) \n(found in $( which pip ))."

CMD ["python3"]
