---
title: "Apktools原理分析"
date: 2019-05-25T22:12:59+08:00
tag : "Rom"
category : "安卓基础"
description : "本文主要是Apktools原理分析"
---

# apktools源码

```
https://github.com/brutall/brut.apktool.git
```

# 阅读顺序

最基础的入口就是没有任何后缀的brut.apktool中的入口函数

```
public static void main(String[] args) throws IOException,
			InterruptedException, BrutException {
		try {
			Verbosity verbosity = Verbosity.NORMAL;
			int i;
			for (i = 0; i < args.length; i++) {
				String opt = args[i];

				if (opt.startsWith("--version") || (opt.startsWith("-version"))) {
					version_print();
					System.exit(1);
				}
				if (!opt.startsWith("-")) {
					break;
				}
				if ("-v".equals(opt) || "--verbose".equals(opt)) {
					if (verbosity != Verbosity.NORMAL) {
						throw new InvalidArgsError();
					}
					verbosity = Verbosity.VERBOSE;
				} else if ("-q".equals(opt) || "--quiet".equals(opt)) {
					if (verbosity != Verbosity.NORMAL) {
						throw new InvalidArgsError();
					}
					verbosity = Verbosity.QUIET;
				} else {
					throw new InvalidArgsError();
				}
			}
			setupLogging(verbosity);

			if (args.length <= i) {
				throw new InvalidArgsError();
			}
			String cmd = args[i];
			args = Arrays.copyOfRange(args, i + 1, args.length);

			if ("d".equals(cmd) || "decode".equals(cmd)) {
				cmdDecode(args);
			} else if ("b".equals(cmd) || "build".equals(cmd)) {
				cmdBuild(args);
			} else if ("if".equals(cmd) || "install-framework".equals(cmd)) {
				cmdInstallFramework(args);
			} else if ("publicize-resources".equals(cmd)) {
				cmdPublicizeResources(args);
			} else {
				throw new InvalidArgsError();
			}
		} catch (InvalidArgsError ex) {
			usage();
			System.exit(1);
		}
	}
```

从这一段代码中可以看出apktools的一些命令

> -d 反编译

> -b 回编译

> -if 安装框架

> -publicize-resources 处理arsc文件

## 反编译过程

```
private static void cmdDecode(String[] args) throws InvalidArgsError,
			AndrolibException {
		ApkDecoder decoder = new ApkDecoder();

		int i;
		for (i = 0; i < args.length; i++) {
			String opt = args[i];
			if (!opt.startsWith("-")) {
				break;
			}
			if ("-s".equals(opt) || "--no-src".equals(opt)) {
				decoder.setDecodeSources(ApkDecoder.DECODE_SOURCES_NONE);
			} else if ("-d".equals(opt) || "--debug".equals(opt)) {
				decoder.setDebugMode(true);
			} else if ("-b".equals(opt) || "--no-debug-info".equals(opt)) {
				decoder.setBaksmaliDebugMode(false);
			} else if ("-t".equals(opt) || "--frame-tag".equals(opt)) {
				i++;
				if (i >= args.length) {
					throw new InvalidArgsError();
				}
				decoder.setFrameworkTag(args[i]);
			} else if ("-f".equals(opt) || "--force".equals(opt)) {
				decoder.setForceDelete(true);
			} else if ("-r".equals(opt) || "--no-res".equals(opt)) {
				decoder.setDecodeResources(ApkDecoder.DECODE_RESOURCES_NONE);
			} else if ("--keep-broken-res".equals(opt)) {
				decoder.setKeepBrokenResources(true);
			} else if ("--frame-path".equals(opt)) {
				i++;
        if (i >= args.length) {
          throw new InvalidArgsError();
        }
				decoder.setFrameworkDir(args[i]);
			} else {
				throw new InvalidArgsError();
			}
		}

		String outName = null;
		if (args.length == i + 2) {
			outName = args[i + 1];
		} else if (args.length == i + 1) {
			outName = args[i];
			outName = outName.endsWith(".apk") ? outName.substring(0,
					outName.length() - 4) : outName + ".out";
			outName = new File(outName).getName();
		} else {
			throw new InvalidArgsError();
		}
		File outDir = new File(outName);
		decoder.setOutDir(outDir);
		decoder.setApkFile(new File(args[i]));

		try {
			decoder.decode();
		} catch (OutDirExistsException ex) {
			System.out
					.println("Destination directory ("
							+ outDir.getAbsolutePath()
							+ ") "
							+ "already exists. Use -f switch if you want to overwrite it.");
			System.exit(1);
		} catch (InFileNotFoundException ex) {
			System.out.println("Input file (" + args[i] + ") "
					+ "was not found or was not readable.");
			System.exit(1);
		} catch (CantFindFrameworkResException ex) {
			System.out
					.println("Can't find framework resources for package of id: "
							+ String.valueOf(ex.getPkgId())
							+ ". You must install proper "
							+ "framework files, see project website for more info.");
			System.exit(1);
		} catch (IOException ex) {
			System.out
					.println("Could not modify file. Please ensure you have permission.");
			System.exit(1);
		}

	}
```

反编译的过程操纵了一个叫做apkdecoder的类，通过传入的参数进行了一些配置项的更改，之后核心在于

```
        decoder.setApkFile(new File(args[i]));
		try {
			decoder.decode();
		} catch (OutDirExistsException ex) {
		    ...
		}
```

### apkDecoder.decode()


```
public void decode() throws AndrolibException, IOException {
		File outDir = getOutDir();

		if (!mForceDelete && outDir.exists()) {
			throw new OutDirExistsException();
		}

		if (!mApkFile.isFile() || !mApkFile.canRead()) {
			throw new InFileNotFoundException();
		}

		try {
			OS.rmdir(outDir);
		} catch (BrutException ex) {
			throw new AndrolibException(ex);
		}
		outDir.mkdirs();

		if (hasSources()) {
			switch (mDecodeSources) {
			case DECODE_SOURCES_NONE:
				mAndrolib.decodeSourcesRaw(mApkFile, outDir, mDebug);
				break;
			case DECODE_SOURCES_SMALI:
				mAndrolib.decodeSourcesSmali(mApkFile, outDir, mDebug, mBakDeb);
				break;
			case DECODE_SOURCES_JAVA:
				mAndrolib.decodeSourcesJava(mApkFile, outDir, mDebug);
				break;
			}
		}

		if (hasResources()) {

			// read the resources.arsc checking for STORED vs DEFLATE
			// compression
			// this will determine whether we compress on rebuild or not.
			JarFile jf = new JarFile(mApkFile.getAbsoluteFile());
			JarEntry je = jf.getJarEntry("resources.arsc");
			if (je != null) {
				int compression = je.getMethod();
				mCompressResources = (compression != ZipEntry.STORED)
						&& (compression == ZipEntry.DEFLATED);
			}
			jf.close();

			switch (mDecodeResources) {
			case DECODE_RESOURCES_NONE:
				mAndrolib.decodeResourcesRaw(mApkFile, outDir);
				break;
			case DECODE_RESOURCES_FULL:
				mAndrolib.decodeResourcesFull(mApkFile, outDir, getResTable());
				break;
			}
		} else {
			// if there's no resources.asrc, decode the manifest without looking
			// up attribute references
			if (hasManifest()) {
				switch (mDecodeResources) {
				case DECODE_RESOURCES_NONE:
					mAndrolib.decodeManifestRaw(mApkFile, outDir);
					break;
				case DECODE_RESOURCES_FULL:
					mAndrolib.decodeManifestFull(mApkFile, outDir,
							getResTable());
					break;
				}
			}
		}

		mAndrolib.decodeRawFiles(mApkFile, outDir);
		writeMetaFile();
	}
```

这里写的其实蛮清楚的

创建存储的文件夹->解码dex文件到存储的文件夹->处理资源文件->拷贝manifest到存储的文件夹->拷贝raw文件（asset，lib）到存储文件夹->整合metainfo文件到存储的文件夹

也就是解码其实只用到了一处，其余部分都是拷贝或者整合。

其中decode又分为decode为raw文件，decode为smali文件，和decode为java文件三种

默认是decode为smali文件。

raw文件的decode就是直接复制，不解码的意思。

因此我们需要看一下decode为smali和decode为java2种。

#### mAndrolib.decodeSourcesSmali(mApkFile, outDir, mDebug, mBakDeb)

这个参数第四个是指控制debug输出，和第三个一样

```
	public void decodeSourcesSmali(File apkFile, File outDir, boolean debug,
			boolean bakdeb) throws AndrolibException {
		try {
			File smaliDir = new File(outDir, SMALI_DIRNAME);
			OS.rmdir(smaliDir);
			smaliDir.mkdirs();
			LOGGER.info("Baksmaling...");
			SmaliDecoder.decode(apkFile, smaliDir, debug, bakdeb);
		} catch (BrutException ex) {
			throw new AndrolibException(ex);
		}
	}

```

这里也就是创建了文件夹，然后调用了SmaliDecoder的decode方式

而smalidecoder又是调用了baksmali.disassembleDexFile

baksmali的这个api虽然写的长，但是可以分析的，具体源码如下

```
public static void disassembleDexFile(String dexFilePath, DexFile dexFile, boolean deodex, String outputDirectory,
                                          String[] classPathDirs, String bootClassPath, String extraBootClassPath,
                                          boolean noParameterRegisters, boolean useLocalsDirective,
                                          boolean useSequentialLabels, boolean outputDebugInfo, boolean addCodeOffsets,
                                          boolean noAccessorComments, int registerInfo, boolean verify,
                                          boolean ignoreErrors, String inlineTable, boolean checkPackagePrivateAccess)
    {
        baksmali.noParameterRegisters = noParameterRegisters;
        baksmali.useLocalsDirective = useLocalsDirective;
        baksmali.useSequentialLabels = useSequentialLabels;
        baksmali.outputDebugInfo = outputDebugInfo;
        baksmali.addCodeOffsets = addCodeOffsets;
        baksmali.noAccessorComments = noAccessorComments;
        baksmali.deodex = deodex;
        baksmali.registerInfo = registerInfo;
        baksmali.bootClassPath = bootClassPath;
        baksmali.verify = verify;

        if (registerInfo != 0 || deodex || verify) {//0 || false || false
            try {
                String[] extraBootClassPathArray = null;
                if (extraBootClassPath != null && extraBootClassPath.length() > 0) {
                    assert extraBootClassPath.charAt(0) == ':';
                    extraBootClassPathArray = extraBootClassPath.substring(1).split(":");
                }

                if (dexFile.isOdex() && bootClassPath == null) {
                    //ext.jar is a special case - it is typically the 2nd jar in the boot class path, but it also
                    //depends on classes in framework.jar (typically the 3rd jar in the BCP). If the user didn't
                    //specify a -c option, we should add framework.jar to the boot class path by default, so that it
                    //"just works"
                    if (extraBootClassPathArray == null && isExtJar(dexFilePath)) {
                        extraBootClassPathArray = new String[] {"framework.jar"};
                    }
                    ClassPath.InitializeClassPathFromOdex(classPathDirs, extraBootClassPathArray, dexFilePath, dexFile,
                            checkPackagePrivateAccess);
                } else {
                    String[] bootClassPathArray = null;
                    if (bootClassPath != null) {
                        bootClassPathArray = bootClassPath.split(":");
                    }
                    ClassPath.InitializeClassPath(classPathDirs, bootClassPathArray, extraBootClassPathArray,
                            dexFilePath, dexFile, checkPackagePrivateAccess);
                }

                if (inlineTable != null) {
                    inlineResolver = new CustomInlineMethodResolver(inlineTable);
                }
            } catch (Exception ex) {
                System.err.println("\n\nError occured while loading boot class path files. Aborting.");
                ex.printStackTrace(System.err);
                System.exit(1);
            }
        }

        File outputDirectoryFile = new File(outputDirectory);
        if (!outputDirectoryFile.exists()) {
            if (!outputDirectoryFile.mkdirs()) {
                System.err.println("Can't create the output directory " + outputDirectory);
                System.exit(1);
            }
        }

        if (!noAccessorComments) {//!false
            syntheticAccessorResolver = new SyntheticAccessorResolver(dexFile);
        }

        //sort the classes, so that if we're on a case-insensitive file system and need to handle classes with file
        //name collisions, then we'll use the same name for each class, if the dex file goes through multiple
        //baksmali/smali cycles for some reason. If a class with a colliding name is added or removed, the filenames
        //may still change of course
        ArrayList<ClassDefItem> classDefItems = new ArrayList<ClassDefItem>(dexFile.ClassDefsSection.getItems());
        Collections.sort(classDefItems, new Comparator<ClassDefItem>() {
            public int compare(ClassDefItem classDefItem1, ClassDefItem classDefItem2) {
                return classDefItem1.getClassType().getTypeDescriptor().compareTo(classDefItem1.getClassType().getTypeDescriptor());
            }
        });

        ClassFileNameHandler fileNameHandler = new ClassFileNameHandler(outputDirectoryFile, ".smali");

        for (ClassDefItem classDefItem: classDefItems) {
            /**
             * The path for the disassembly file is based on the package name
             * The class descriptor will look something like:
             * Ljava/lang/Object;
             * Where the there is leading 'L' and a trailing ';', and the parts of the
             * package name are separated by '/'
             */

            String classDescriptor = classDefItem.getClassType().getTypeDescriptor();

            //validate that the descriptor is formatted like we expect
            if (classDescriptor.charAt(0) != 'L' ||
                classDescriptor.charAt(classDescriptor.length()-1) != ';') {
                System.err.println("Unrecognized class descriptor - " + classDescriptor + " - skipping class");
                continue;
            }

            File smaliFile = fileNameHandler.getUniqueFilenameForClass(classDescriptor);

            //create and initialize the top level string template
            ClassDefinition classDefinition = new ClassDefinition(classDefItem);

            //write the disassembly
            Writer writer = null;
            try
            {
                File smaliParent = smaliFile.getParentFile();
                if (!smaliParent.exists()) {
                    if (!smaliParent.mkdirs()) {
                        System.err.println("Unable to create directory " + smaliParent.toString() + " - skipping class");
                        continue;
                    }
                }

                if (!smaliFile.exists()){
                    if (!smaliFile.createNewFile()) {
                        System.err.println("Unable to create file " + smaliFile.toString() + " - skipping class");
                        continue;
                    }
                }

                BufferedWriter bufWriter = new BufferedWriter(new OutputStreamWriter(
                        new FileOutputStream(smaliFile), "UTF8"));

                writer = new IndentingWriter(bufWriter);
                classDefinition.writeTo((IndentingWriter)writer);
            } catch (Exception ex) {
                System.err.println("\n\nError occured while disassembling class " + classDescriptor.replace('/', '.') + " - skipping class");
                ex.printStackTrace();
                smaliFile.delete();
            }
            finally
            {
                if (writer != null) {
                    try {
                        writer.close();
                    } catch (Throwable ex) {
                        System.err.println("\n\nError occured while closing file " + smaliFile.toString());
                        ex.printStackTrace();
                    }
                }
            }

            if (!ignoreErrors && classDefinition.hadValidationErrors()) {
                System.exit(1);
            }
        }
    }
```

流程精简如下

```
从dex文件中拿出class到集合类中

根据typedescriptor对class进行排序

对描述符为object的class文件进行处理，将处理的结果写入根据class名新建的smali文件中

```

上面所说的处理过程如下

```
public void writeTo(IndentingWriter writer) throws IOException {
        writeClass(writer);
        writeSuper(writer);
        writeSourceFile(writer);
        writeInterfaces(writer);
        writeAnnotations(writer);
        writeStaticFields(writer);
        writeInstanceFields(writer);
        writeDirectMethods(writer);
        writeVirtualMethods(writer);
    }
```

这里看的很仔细了，无论是类信息，父类，源文件等等，全都写入了smali

到这里就完成了dex文件转为smali文件的decode过程

#### mAndrolib.decodeSourcesJava(mApkFile, outDir, mDebug);

好，让我们来学习以下怎么decode为java文件的

```
public void decodeSourcesJava(ExtFile apkFile, File outDir, boolean debug)
			throws AndrolibException {
		LOGGER.info("Decoding Java sources...");
		new AndrolibJava().decode(apkFile, outDir);
	}
```

嗯。。。调用了androidlibjava的decode，这个decode写了什么呢？

```
public void decode(ExtFile apkFile, File outDir) {
		throw new UnsupportedOperationException("Not yet implemented");
	}
```

坑爹啊，原来没实现！！！

所以其实只有两种方式，一种是直接拷贝，二是将dex转为smali，而将dex转为java的还没有写出来！

## 回编译过程

//TODO ：19。6。2
