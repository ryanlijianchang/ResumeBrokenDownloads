# ResumeBrokenDownloads

Java实现断点下载，在下载未完成前断开连接后，重新下载时会在原来的基础上继续下载。


    import java.io.BufferedInputStream;
	import java.io.File;
	import java.io.FileInputStream;
	import java.io.RandomAccessFile;
	import java.net.HttpURLConnection;
	import java.net.URL;
	
	public class ResumeDownload {
		public static final String DOWNLOAD_URL = "http://7xs0af.com1.z0.glb.clouddn.com/High-Wake.mp3";
		public static final String DOWNLOAD_PARENT_PATH = "D:\\test_resume_download\\hi";
		public static final int THREAD_COUNT = 3;
	
		public static void main(String[] args) {
			try {
				// 获取到下载地址的连接
				URL mUrl = new URL(DOWNLOAD_URL);
				HttpURLConnection conn = (HttpURLConnection) mUrl.openConnection();
				// 获取下载文件的大小
				int fileLen = conn.getContentLength();
				// 通过下载链接获取下载文件的文件名
				String filePathUrl = conn.getURL().getFile();
				String fileName = filePathUrl.substring(filePathUrl.lastIndexOf(File.separator) + 1);
				// 生成下载路径
				String fileDownloadPath = DOWNLOAD_PARENT_PATH + File.separator + fileName;
				// 判断父路径是否存在，不存在就生成
				File file = new File(fileDownloadPath);
				if (!file.getParentFile().exists()) {
					file.getParentFile().mkdirs();
				}
				// 关闭连接
				conn.disconnect();
	
				/**
				 * 以下为多线程下载，主要原理就是将文件大小均分多块（根据线程数） 每一个线程从不同的起始位置，下载相等大小的文件 主要通过
				 * HttpUrlConnection里面设置Range参数，设置每一个线程下载的范围
				 * setRequestProperty("Range", "bytes=" + startPos + "-" + endPos);
				 */
	
				int blockSize = fileLen / THREAD_COUNT;
				for (int threadId = 1; threadId <= THREAD_COUNT; threadId++) {
					// 获取每一个线程下载的起始位置和结束位置
					long startPos = (threadId - 1) * blockSize;
					long endPos = threadId * blockSize - 1;
					if (threadId == THREAD_COUNT) {
						endPos = fileLen;
					}
	
					// 然后通过再不同线程里面实现下载逻辑
					// 具体实现在DownloadThread这个Runnable里面
					new Thread(new DownLoadTask(threadId, startPos, endPos, fileDownloadPath, DOWNLOAD_URL)).start();
				}
	
			} catch (Exception e) {
				e.printStackTrace();
			}
	
		}
	}
	
	/**
	 * 具体下载逻辑
	 * 
	 * @author Administrator
	 *
	 */
	class DownLoadTask implements Runnable {
		public static final String TEMP_NAME = "_tempfile";
		private int threadId; // 当前线程id
		private long startPos; // 下载的起始位置
		private long endPos; // 下载的结束位置
		private String fileDownloadPath; // 下载文件存放的文件位置
		private String downloadUrl; // 下载链接
	
		private String tempFilePath; // 记录进度的临时文件路径
	
		public DownLoadTask(int threadId, long startPos, long endPos, String fileDownloadPath, String downloadUrl) {
			super();
			this.threadId = threadId;
			this.startPos = startPos;
			this.endPos = endPos;
			this.fileDownloadPath = fileDownloadPath;
			this.downloadUrl = downloadUrl;
	
			this.tempFilePath = fileDownloadPath + TEMP_NAME + threadId;
		}
	
		@Override
		public void run() {
			try {
				// 记录下载的开始时间
				long startTime = System.currentTimeMillis();
	
				URL mUrl = new URL(downloadUrl);
	
				// 为了实现断点下载，在重新下载时从缓存文件里面获取下载的起始位置
				if (getProgress(threadId) != 0) {
					startPos = getProgress(threadId);
				}
	
				System.out.println("线程" + threadId + "继续下载，开始位置：" + startPos + "结束位置是：" + endPos);
	
				// HttpUrlConnection的常规操作
				// 要实现断点下载的话，必须要设置mConnection.setRequestProperty("Range", "bytes=" +
				// startPos + "-" + endPos);
				HttpURLConnection mConnection = (HttpURLConnection) mUrl.openConnection();
				mConnection.setRequestMethod("POST");
				mConnection.setReadTimeout(5000);
				mConnection.setRequestProperty("Charset", "UTF-8");
				mConnection.setRequestProperty("Range", "bytes=" + startPos + "-" + endPos);
				mConnection.connect();
	
				// 如果下载路径不存在的话，则创建文件路径
				File file = new File(fileDownloadPath);
				if (!file.getParentFile().exists()) {
					file.getParentFile().mkdirs();
				}
	
				// 通过RandomAccessFile对要下载的文件进行读写
				RandomAccessFile downloadFile = new RandomAccessFile(fileDownloadPath, "rw");
				// 写的时候，将光标移到要下载的起始位置
				downloadFile.seek(startPos);
	
				BufferedInputStream bis = new BufferedInputStream(mConnection.getInputStream());
				int size = 0; // 获取缓存区存放的字节大小
				long len = 0; // 记录本次下载的大小，以便计算本次下载的起始位置移动到了哪里
				byte[] buf = new byte[1024];
				while ((size = bis.read(buf)) != -1) {
					// 累加
					len += size;
					// 然后将缓冲区的内容写到下载文件中
					downloadFile.write(buf, 0, size);
					// 然后将下载的起始位置移动到已经下载完的末尾，写到缓存文件里面去
					setProgress(threadId, startPos + len);
				}
	
				// 获取下载结束时间，输出
				long curTime = System.currentTimeMillis();
				System.out.println("线程" + threadId + "已经下载完成，耗时：" + (curTime - startTime) + "ms.");
	
				// 关闭流、文件和连接
				downloadFile.close();
				mConnection.disconnect();
				bis.close();
	
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	
		/**
		 * 从temp文件获取下载进度
		 * 
		 * @param threadId
		 * @return
		 */
		private long getProgress(int threadId) {
			try {
				File markFile = new File(tempFilePath);
				if (!markFile.exists()) {
					return 0;
				}
				FileInputStream fis = new FileInputStream(markFile);
				BufferedInputStream bis = new BufferedInputStream(fis);
				byte[] buf = new byte[1024];
				String startPos = "";
				int len = -1;
				while ((len = bis.read(buf)) != -1) {
					startPos += new String(buf, 0, len);
				}
	
				// 不关闭流的话，不能删除文件
				fis.close();
				bis.close();
	
				return Long.parseLong(startPos);
	
			} catch (Exception e) {
				e.printStackTrace();
			}
			return 0;
		}
	
		/**
		 * 在temp文件记录下载进度
		 * 
		 * @param threadId
		 * @param startPos
		 */
		private void setProgress(int threadId, long startPos) {
			try {
				File markFile = new File(tempFilePath);
				if (!markFile.getParentFile().exists()) {
					markFile.getParentFile().mkdirs();
				}
				
				RandomAccessFile rr = new RandomAccessFile(markFile, "rw");// 存储下载标记的文件
				String strStartPos = String.valueOf(startPos);
				rr.write(strStartPos.getBytes(), 0, strStartPos.length());
				
				rr.close();
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				// 当文件下载完成时，即开始位置和结束位置重合时，删除记录进度的缓存文件
				if (startPos >= endPos) {
					File markFile = new File(tempFilePath);
					if (markFile.exists()) {
						System.out.println("markFile delete");
						markFile.delete();
					}
				}
			}
	
		}
	}

