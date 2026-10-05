CXX      ?= g++
CXXFLAGS ?= -std=c++17 -O2 -Wall -Wextra

TARGET := container-forge
SOURCE := container_forge.cpp

.PHONY: all rootfs clean clean-all

all: $(TARGET)

$(TARGET): $(SOURCE)
	$(CXX) $(CXXFLAGS) -o $@ $<

rootfs: make_rootfs.sh
	sudo ./make_rootfs.sh ./rootfs

clean:
	rm -f $(TARGET)

clean-all: clean
	sudo rm -rf rootfs
