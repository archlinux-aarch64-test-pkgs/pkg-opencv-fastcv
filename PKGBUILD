# Maintainer: Antonio Rojas <arojas@archlinux.org>
# Contributor: Ray Rashif <schiv@archlinux.org>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>

pkgbase=opencv
pkgname=(opencv
         opencv-samples
         python-opencv
        #  opencv-cuda
        #  python-opencv-cuda
         opencv-fastcv
         python-opencv-fastcv
         opencv-fastcv-tests)
pkgver=4.13.0
pkgrel=2
pkgdesc='Open Source Computer Vision Library'
arch=('aarch64')
license=(Apache-2.0)
url='https://opencv.org/'
depends=(abseil-cpp
         cblas
         ffmpeg
         freetype2
         gcc-libs
         glib2
         glibc
         gst-plugins-base
         gst-plugins-base-libs
         gstreamer
         harfbuzz
         lapack
         libdc1394
         libglvnd
         libjpeg-turbo
         libjxl
         libpng
         libtiff
         libwebp
         openexr
         openjpeg2
         verdict
         protobuf
         tbb
         zlib)
makedepends=(ant
             cmake
            #  cuda
            #  cudnn
             eigen
             fmt
             git
             glew
             hdf5
             java-environment
             lapacke
             mesa
             nlohmann-json
             openmpi
             pugixml
             python-numpy
             python-setuptools
             qt6-5compat
             vtk)
optdepends=('opencv-samples: samples'
            'vtk: for the viz module'
            'glew: for the viz module'
            'qt6-base: for the HighGUI module'
            'hdf5: for the HDF5 module'
            'opencl-icd-loader: For coding with OpenCL'
            'java-runtime: Java interface')
source=(git+https://github.com/opencv/opencv#tag=$pkgver
        git+https://github.com/opencv/opencv_contrib#tag=$pkgver
        vtk9.patch
        fix-cuda-flags.patch
        fix-cudacodec-dependencies.patch)
sha256sums=('efc754fc3888944007586133fc225f0136d1767020564b81e4d90f51e9d68b60'
            '3fc521a16314978de02d5b33e657a09a9567429d5801d3fb94e35581ea44d729'
            'f35a2d4ea0d6212c7798659e59eda2cb0b5bc858360f7ce9c696c77d3029668e'
            '95472ecfc2693c606f0dd50be2f012b4d683b7b0a313f51484da4537ab8b2bfe'
            'fbb10b75ca7849f85ea2f118aa017f00e34445d80ed76619f13ae1e4e9504ae4')
options=(!lto) # https://gitlab.archlinux.org/archlinux/packaging/packages/kdenlive/-/issues/8

prepare() {
  pushd opencv
  patch -p1 < ../vtk9.patch # Don't require all vtk optdepends
  # https://github.com/opencv/opencv/issues/27223
  # https://bugreports.qt.io/browse/QTBUG-134774
  sed -i 's/add_definitions(${Qt${QT_VERSION_MAJOR}${dt_dep}_DEFINITIONS})/link_libraries(${Qt${QT_VERSION_MAJOR}${dt_dep}})/' modules/highgui/CMakeLists.txt
  # OpenCV passes all CXXFLAGS to nvcc through -Xcompiler, which does not work for '-Wp,something' flags
  # We remove the -Xcompiler and pass our CXXFLAGS through cmake's CUDAFLAGS
  patch -p1 < ../fix-cuda-flags.patch
  popd

  pushd opencv_contrib
  patch -p1 -i ../fix-cudacodec-dependencies.patch # https://github.com/opencv/opencv_contrib/issues/4045
}

build() {
  export JAVA_HOME="/usr/lib/jvm/default"
  local cmake_options=(
    -DWITH_OPENCL=ON
    -DWITH_OPENGL=ON
    -DOpenGL_GL_PREFERENCE=LEGACY
    -DCMAKE_CXX_STANDARD=17
    -DWITH_TBB=ON
    -DWITH_VULKAN=ON
    -DWITH_QT=ON
    -DWITH_JPEGXL=ON
    -DBUILD_TESTS=OFF
    -DBUILD_PERF_TESTS=OFF
    -DBUILD_EXAMPLES=ON
    -DBUILD_PROTOBUF=OFF
    -DPROTOBUF_UPDATE_FILES=ON
    -DINSTALL_C_EXAMPLES=ON
    -DINSTALL_PYTHON_EXAMPLES=ON
    -DCMAKE_INSTALL_PREFIX=/usr
    # -DCPU_BASELINE_DISABLE=SSE3
    # -DCPU_BASELINE_REQUIRE=SSE2
    -DOPENCV_EXTRA_MODULES_PATH="$srcdir"/opencv_contrib/modules
    -DOPENCV_SKIP_PYTHON_LOADER=ON
    # cmake's FindLAPACK doesn't add cblas to LAPACK_LIBRARIES, so we need to specify them manually
    -DLAPACK_LIBRARIES="/usr/lib/liblapack.so;/usr/lib/libblas.so;/usr/lib/libcblas.so"
    -DLAPACK_CBLAS_H=/usr/include/cblas.h
    -DLAPACK_LAPACKE_H=/usr/include/lapacke.h
    -DOPENCV_GENERATE_PKGCONFIG=ON
    -DOPENCV_ENABLE_NONFREE=ON
    -DOPENCV_JNI_INSTALL_PATH=lib
    -DOPENCV_GENERATE_SETUPVARS=OFF
    -DEIGEN_INCLUDE_PATH=/usr/include/eigen3
    -Dprotobuf_MODULE_COMPATIBLE=ON
    -DHDF5_NO_FIND_PACKAGE_CONFIG_FILE=ON
  )

  # Clear cached FastCV variables to work around a re-configuration bug in OpenCV's cmake:
  # on non-clean builds, cached FastCV_INCLUDE_PATH/FastCV_LIB_PATH cause the "external" code
  # path to be taken, which doesn't create the imported target that FASTCV_LIBRARY="fastcv" expects.
  cmake -B build-fastcv -UFastCV_INCLUDE_PATH -UFastCV_LIB_PATH -UFASTCV_LIBRARY -UHAVE_FASTCV \
    -S $pkgname "${cmake_options[@]}" \
    -DBUILD_WITH_DEBUG_INFO=OFF \
    -DWITH_FASTCV=ON
  cmake --build build-fastcv

  # Build FastCV unit tests and perf tests
  cmake -B build-fastcv -S $pkgname -UFastCV_INCLUDE_PATH -UFastCV_LIB_PATH -UFASTCV_LIBRARY -UHAVE_FASTCV \
    "${cmake_options[@]}" -DBUILD_WITH_DEBUG_INFO=OFF -DWITH_FASTCV=ON \
    -DBUILD_TESTS=ON -DBUILD_PERF_TESTS=ON
  cmake --build build-fastcv --target opencv_test_fastcv opencv_perf_fastcv

  cmake -B build -S $pkgname "${cmake_options[@]}" \
    -DBUILD_WITH_DEBUG_INFO=ON
  cmake --build build

  # In general, we want to list all real archs (sm_XX) and the latest virtual arch (compute_XX) for future PTX compatibility.
  # Valid values can be discovered from nvcc --help
  # local cuda_archs="75;80;86;87;88;89;90;100;103;110;120;121;121-virtual"

  # Avoid nvcc intercepting -Werror=format-security: Value 'format-security' is not defined for option 'Werror'
  # CUDAFLAGS="${CXXFLAGS/-Werror=format-security/-Xcompiler -Werror=format-security} -fno-lto --threads 0" \
  # CFLAGS="${CFLAGS} -fno-lto" CXXFLAGS="${CXXFLAGS} -fno-lto" LDFLAGS="${LDFLAGS} -fno-lto" \
  # cmake -B build-cuda -S $pkgname "${cmake_options[@]}" \
  #   -DBUILD_WITH_DEBUG_INFO=OFF \
  #   -DWITH_CUDA=ON \
  #   -DWITH_CUDNN=ON \
  #   -DENABLE_CUDA_FIRST_CLASS_LANGUAGE=ON \
  #   -DCMAKE_CUDA_ARCHITECTURES="$cuda_archs"
  # cmake --build build-cuda --verbose
}

package_opencv() {
  DESTDIR="$pkgdir" cmake --install build

  # separate samples package
  mv "$pkgdir"/usr/share/opencv4/samples "$srcdir"

  # Add java symlinks expected by some binary blobs
  ln -sr "$pkgdir"/usr/share/java/{opencv4/opencv-${pkgver//./},opencv}.jar
  ln -sr "$pkgdir"/usr/lib/{libopencv_java${pkgver//./},libopencv_java}.so

  # Split Python bindings
  rm -r "$pkgdir"/usr/lib/python3*
}

package_opencv-samples() {
  pkgdesc+=' (samples)'
  depends=(opencv)
  unset optdepends

  mkdir -p "$pkgdir"/usr/share/opencv4
  mv samples "$pkgdir"/usr/share/opencv4
}

package_python-opencv() {
  pkgdesc='Python bindings for OpenCV'
  depends=(fmt
           glew
           hdf5
           jsoncpp
           opencv
           openmpi
           pugixml
           python-numpy
           qt6-base
           vtk)
  unset optdepends

  DESTDIR="$pkgdir" cmake --install build/modules/python3
}

# package_opencv-cuda() {
#   pkgdesc+=' (with CUDA support)'
#   depends+=(cudnn)
#   conflicts=(opencv opencv-fastcv)
#   provides=(opencv=$pkgver)
#   options=(!debug)

#   DESTDIR="$pkgdir" cmake --install build-cuda

#   # Split samples
#   rm -r "$pkgdir"/usr/share/opencv4/samples

#   # Add java symlinks expected by some binary blobs
#   ln -sr "$pkgdir"/usr/share/java/{opencv4/opencv-${pkgver//./},opencv}.jar
#   ln -sr "$pkgdir"/usr/lib/{libopencv_java${pkgver//./},libopencv_java}.so

#   # Split Python bindings
#   rm -r "$pkgdir"/usr/lib/python3*
# }

# package_python-opencv-cuda() {
#   pkgdesc='Python bindings for OpenCV (with CUDA support)'
#   depends=(fmt
#            glew
#            hdf5
#            jsoncpp
#            opencv-cuda
#            openmpi
#            pugixml
#            python-numpy
#            qt6-base
#            vtk)
#   conflicts=(python-opencv python-opencv-fastcv)
#   provides=(python-opencv=$pkgver)
#   unset optdepends

#   DESTDIR="$pkgdir" cmake --install build-cuda/modules/python3
# }

package_opencv-fastcv() {
  pkgdesc+=' (with Qualcomm FastCV support)'
  depends+=(qcom-fastrpc qcom-fastcv-binaries)
  conflicts=(opencv opencv-cuda)
  provides=(opencv=$pkgver)
  options=(!debug)

  DESTDIR="$pkgdir" cmake --install build-fastcv

  # Split samples
  rm -r "$pkgdir"/usr/share/opencv4/samples

  # Add java symlinks expected by some binary blobs
  ln -sr "$pkgdir"/usr/share/java/{opencv4/opencv-${pkgver//./},opencv}.jar
  ln -sr "$pkgdir"/usr/lib/{libopencv_java${pkgver//./},libopencv_java}.so

  # Split Python bindings
  rm -r "$pkgdir"/usr/lib/python3*
}

package_python-opencv-fastcv() {
  pkgdesc='Python bindings for OpenCV (with Qualcomm FastCV support)'
  depends=(fmt
           glew
           hdf5
           jsoncpp
           opencv-fastcv
           openmpi
           pugixml
           python-numpy
           qt6-base
           vtk)
  conflicts=(python-opencv python-opencv-cuda)
  provides=(python-opencv=$pkgver)
  unset optdepends

  DESTDIR="$pkgdir" cmake --install build-fastcv/modules/python3
}

package_opencv-fastcv-tests() {
  pkgdesc='Unit tests and perf tests for OpenCV FastCV module'
  depends=(opencv-fastcv)
  unset optdepends

  install -Dm755 build-fastcv/bin/opencv_test_fastcv "$pkgdir"/usr/bin/opencv_test_fastcv
  install -Dm755 build-fastcv/bin/opencv_perf_fastcv "$pkgdir"/usr/bin/opencv_perf_fastcv
}
