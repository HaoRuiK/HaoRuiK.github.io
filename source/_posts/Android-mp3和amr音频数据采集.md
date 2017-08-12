---
title: Mp3和Amr音频数据采集
date: 2016-11-13 22:07:43
tags: Android

---

#### 需求：

- 使用一份音频输入数据分别采样Mp3和Amr音频编码需要的数据。

#### 前提条件：

- Mp3文件需要采样率是44.1KHZ的音频数据流
- Amr文件需要采样率是8,000 Hz或者16,000 HZ采用率的音频数据流


#### 具体实现：

- 创建音频采集线程，采样音频数据

<!--more-->

```java
public class Recorder implements Runnable{
    
  	private ArrayList<Integer> hitArray; 
    
    @Override
    public void start() {
        if (isRecording) {
            Log.e(TAG, "is recoding");
            return;
        }
      	//获取指定格式的音频缓冲区数据大小
      	//指定采样率为44100，单声道，PCM16位编码方式
        int bufferSizeInByte = AudioRecord.getMinBufferSize(SAMPLE_RATE,
                AudioFormat.CHANNEL_CONFIGURATION_MONO,
                AudioFormat.ENCODING_PCM_16BIT);

        audioBuffer = new short[bufferSizeInByte / 2];

		//创建音频采集器，指定音频源为麦克风，其他参数和上面相同
        audioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC,
                SAMPLE_RATE,
                AudioFormat.CHANNEL_CONFIGURATION_MONO,
                AudioFormat.ENCODING_PCM_16BIT,
                bufferSizeInByte);
      	//开始音频采集
        audioRecord.startRecording();
		
        isRecording = true;
      	//初始化Mp3编码器
        mMp3Format = new MP3Format();
        mMp3Format.start(bufferSizeInByte);

        //开启线程
        runningThread = new Thread(this);
        runningThread.start();
      
      	//获取需要取出的short下标List  下文说明
        hitArray = getIndexList();
    }
}
   
```

- **思路**

> 由于amr是码率50fps的。所以每个frame需要的数据源就是8000/50=160个short
>
> 现在对应在44100上就是每个frame 44100/50 = 882 个short
>
> 然后就是从44100采样的每882个short里拿出160个给amr用
>
> 也就是每5.5125个short拿一个short放到amr的Buffer里
>
> 这个Buffer每满160就拿去encode
>
> 然后接着从audioBuffer里拿



- 首先如果要保证声音的连续性就得首先获取882个short中这160个short的下标

```java
private ArrayList<Integer> getIndexList(){
  	float ratio = (float) (882.0 / 160);
  	ArrayList<Integer> indexList = new ArrayList();
	int count = 0;
	for (int i = 1; i <= 882; i++){
		if(i / ratio > count){
          	indexList.add(i);
			count++;
		}
	}
  return indexList;
}

```



- 获取进行编码的数据

```java
 @Override
    public void run() {
        while (isRecording) {
            //每次采集到的音频流数据
            int read = audioRecord.read(audioBuffer, 0, FRAME_SIZE);
          	//判断数据是否正常
            if (read == AudioRecord.ERROR_INVALID_OPERATION || read == AudioRecord.ERROR_BAD_VALUE) {
                Log.i(TAG, "error:" + read);
                continue;
            }
          //这里做Mp3encode
          mMp3Format.formatMp3(mp3AudioBuffer, mp3BufferSize);
          
       		 for (int i = 0; i < read; i++) {
                      for (int i = 0; i < read; i++) {
                            if (sourcePointer == hitArray[hitPointer]) {
                                amrAudioBuffer[targetPointer++] = audioBuffer[i];
                                hitPointer++;
                                if (hitPointer == 160) {
                                    hitPointer = 0;
                                    targetPointer = 0;
                                    //这里做amrencode amrencode(amrBuffer,160);
                                    pcmConsumer.onPcmFeed(amrAudioBuffer, 160);
                                }
                            }
                            if (sourcePointer == 882) {
                                sourcePointer = 0;
                            } else {
                                sourcePointer++;
                            }
                        }
                    }
     
        }    
       
```



- **坑**

> 后来发现一些移动设备并**不支持**44.1KHZ的采样率，在AudioRecord.getMinBufferSize() 方法获取音频缓冲区大小的时候返回负数。
>
> 在测试了多个设备的可以支持的采样率之后，确定采样率为比较通用的8,000 Hz。



- **方案**


- 更改音频采集器采样率为8000 Hz
- 扩充数据，拿取到需要编码的数据

```java
 @Override
    public void run() {
        while (isRecording) {
            //每次采集到的音频流数据
            int read = audioRecord.read(audioBuffer, 0, FRAME_SIZE);
          	//判断数据是否正常
            if (read == AudioRecord.ERROR_INVALID_OPERATION || read == AudioRecord.ERROR_BAD_VALUE) {
                Log.i(TAG, "error:" + read);
                continue;
            }
            int mp3BufferSize = 0;
            for (int i = 0; i < read; i++) {
              	//判断当前的数据需要复制几份
                if (i % 2 == 0) {
                    mp3BufferSize += 6;
                } else {
                    mp3BufferSize += 5;
                }
            }
            //amrencode 
            pcmConsumer.onPcmFeed(audioBuffer, read);
            //创建一个新数组用来存储用来合成Mp3的数据
            mp3AudioBuffer = new short[mp3BufferSize];
            int position = 0;
            for (int i = 0; i < read; i++) {
                //每一个short复制三份填充到MP3Short数组中
                if (i % 2 == 0) {
                    for (int j = 0; j < 6; j++) {
                     
                        mp3AudioBuffer[position] = audioBuffer[i];
                        position++;
                    }
                } else {
                    for (int j = 0; j < 5; j++) {
                        mp3AudioBuffer[position] = audioBuffer[i];
                        position++;
                    }
                }
            }
            //mp3encode
            mMp3Format.formatMp3(mp3AudioBuffer, mp3BufferSize);
        }
    }
```


> MP3编码用的是从mp3lame官网下载的源码然后重新编译的**mp3lame.so**库。
>
> Amr编码使用的是Android自带的**media_jni.so**库