Release Process
====================

* * *

###update (commit) version in sources


	ecent-qt.pro
	contrib/verifysfbinaries/verify.sh
	doc/README*
	share/setup.nsi
	src/clientversion.h (change CLIENT_VERSION_IS_RELEASE to true)

###tag version in git

	git tag -a v0.8.0

###write release notes. git shortlog helps a lot, for example:

	git shortlog --no-merges v0.7.2..v0.8.0

* * *

##perform gitian builds

 From a directory containing the ecent source, gitian-builder and gitian.sigs
  
	export SIGNER=(your gitian key, ie bluematt, sipa, etc)
	export VERSION=0.8.0
	cd ./gitian-builder

 Fetch and build inputs: (first time, or when dependency versions change)

	mkdir -p inputs; cd inputs/
	wget 'http://miniupnp.free.fr/files/download.php?file=miniupnpc-1.6.tar.gz' -O miniupnpc-1.6.tar.gz
	wget 'http://www.openssl.org/source/openssl-1.0.1c.tar.gz'
	wget 'http://download.oracle.com/berkeley-db/db-4.8.30.NC.tar.gz'
	wget 'http://zlib.net/zlib-1.2.6.tar.gz'
	wget 'ftp://ftp.simplesystems.org/pub/libpng/png/src/libpng-1.5.9.tar.gz'
	wget 'http://fukuchi.org/works/qrencode/qrencode-3.2.0.tar.bz2'
	wget 'http://downloads.sourceforge.net/project/boost/boost/1.50.0/boost_1_50_0.tar.bz2'
	wget 'http://releases.qt-project.org/qt4/source/qt-everywhere-opensource-src-4.8.3.tar.gz'
	cd ..
	./bin/gbuild ../ecent/contrib/gitian-descriptors/boost-win32.yml
	mv build/out/boost-win32-1.50.0-gitian2.zip inputs/
	./bin/gbuild ../ecent/contrib/gitian-descriptors/qt-win32.yml
	mv build/out/qt-win32-4.8.3-gitian-r1.zip inputs/
	./bin/gbuild ../ecent/contrib/gitian-descriptors/deps-win32.yml
	mv build/out/ecent-deps-0.0.5.zip inputs/

 Build ecentd and ecent-qt on Linux32, Linux64, and Win32:
  
	./bin/gbuild --commit ecent=v${VERSION} ../ecent/contrib/gitian-descriptors/gitian.yml
	./bin/gsign --signer $SIGNER --release ${VERSION} --destination ../gitian.sigs/ ../ecent/contrib/gitian-descriptors/gitian.yml
	pushd build/out
	zip -r ecent-${VERSION}-linux-gitian.zip *
	mv ecent-${VERSION}-linux-gitian.zip ../../
	popd
	./bin/gbuild --commit ecent=v${VERSION} ../ecent/contrib/gitian-descriptors/gitian-win32.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-win32 --destination ../gitian.sigs/ ../ecent/contrib/gitian-descriptors/gitian-win32.yml
	pushd build/out
	zip -r ecent-${VERSION}-win32-gitian.zip *
	mv ecent-${VERSION}-win32-gitian.zip ../../
	popd

  Build output expected:

  1. linux 32-bit and 64-bit binaries + source (ecent-${VERSION}-linux-gitian.zip)
  2. windows 32-bit binary, installer + source (ecent-${VERSION}-win32-gitian.zip)
  3. Gitian signatures (in gitian.sigs/${VERSION}[-win32]/(your gitian key)/

repackage gitian builds for release as stand-alone zip/tar/installer exe

**Linux .tar.gz:**

	unzip ecent-${VERSION}-linux-gitian.zip -d ecent-${VERSION}-linux
	tar czvf ecent-${VERSION}-linux.tar.gz ecent-${VERSION}-linux
	rm -rf ecent-${VERSION}-linux

**Windows .zip and setup.exe:**

	unzip ecent-${VERSION}-win32-gitian.zip -d ecent-${VERSION}-win32
	mv ecent-${VERSION}-win32/ecent-*-setup.exe .
	zip -r ecent-${VERSION}-win32.zip bitcoin-${VERSION}-win32
	rm -rf ecent-${VERSION}-win32

**Perform Mac build:**

  OSX binaries are created by Gavin Andresen on a 32-bit, OSX 10.6 machine.

	qmake RELEASE=1 USE_UPNP=1 USE_QRCODE=1 ecent-qt.pro
	make
	export QTDIR=/opt/local/share/qt4  # needed to find translations/qt_*.qm files
	T=$(contrib/qt_translations.py $QTDIR/translations src/qt/locale)
	python2.7 share/qt/clean_mac_info_plist.py
	python2.7 contrib/macdeploy/macdeployqtplus Bitcoin-Qt.app -add-qt-tr $T -dmg -fancy contrib/macdeploy/fancy.plist

 Build output expected: Bitcoin-Qt.dmg

###Next steps:

* Code-sign Windows -setup.exe (in a Windows virtual machine) and
  OSX Bitcoin-Qt.app (Note: only Gavin has the code-signing keys currently)

* upload builds to SourceForge

* create SHA256SUMS for builds, and PGP-sign it

* update ecent.org version
  make sure all OS download links go to the right versions

* update forum version

* update wiki download links

* update wiki changelog: [https://en.ecent.it/wiki/Changelog](https://en.bitcoin.it/wiki/Changelog)

Commit your signature to gitian.sigs:

	pushd gitian.sigs
	git add ${VERSION}/${SIGNER}
	git add ${VERSION}-win32/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

-------------------------------------------------------------------------

### After 3 or more people have gitian-built, repackage gitian-signed zips:

From a directory containing ecent source, gitian.sigs and gitian zips

	export VERSION=0.5.1
	mkdir ecent-${VERSION}-linux-gitian
	pushd ecent-${VERSION}-linux-gitian
	unzip ../ecent-${VERSION}-linux-gitian.zip
	mkdir gitian
	cp ../ecent/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}/); do
	 cp ../gitian.sigs/${VERSION}/${signer}/ecent-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}/${signer}/ecent-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r ecent-${VERSION}-linux-gitian.zip *
	cp ecent-${VERSION}-linux-gitian.zip ../
	popd
	mkdir ecent-${VERSION}-win32-gitian
	pushd ecent-${VERSION}-win32-gitian
	unzip ../ecent-${VERSION}-win32-gitian.zip
	mkdir gitian
	cp ../ecent/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}-win32/); do
	 cp ../gitian.sigs/${VERSION}-win32/${signer}/ecent-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}-win32/${signer}/ecent-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r ecent-${VERSION}-win32-gitian.zip *
	cp ecent-${VERSION}-win32-gitian.zip ../
	popd

- Upload gitian zips to SourceForge
- Celebrate 
