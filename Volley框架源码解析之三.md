前面两篇文章介绍了Volley的http请求和缓存部分，其实Volley框架还实现了一个简单的ImageLoader，想了解如何实现一个imageloader的童鞋可以参考下，基本上实现的重要细节都有提到。按照惯例，先给出一个源码的逻辑结构图：<!--more-->
<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201404//imageload2.png"><img class="alignnone size-full wp-image-48" alt="imageload2" src="http://bcs.duapp.com/myblog-wrodpress//blog/201404//imageload2.png" width="1040" height="1120" /></a>
部分源码分析：
1、获取一个ImageListener，这个ImageListener会在出错的时候加载errorImageResId，请求之前先加载默认图，请求正确返回之后加载网络/本地图片。
<pre>/**
     * The default implementation of ImageListener which handles basic functionality
     * of showing a default image until the network response is received, at which point
     * it will switch to either the actual image or the error image.
     * @param imageView The imageView that the listener is associated with.
     * @param defaultImageResId Default image resource ID to use, or 0 if it doesn't exist.
     * @param errorImageResId Error image resource ID to use, or 0 if it doesn't exist.
     */
    public static ImageListener getImageListener(final ImageView view,
            final int defaultImageResId, final int errorImageResId) {
        return new ImageListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                if (errorImageResId != 0) {
                    view.setImageResource(errorImageResId);
                }
            }

            @Override
            public void onResponse(ImageContainer response, boolean isImmediate) {
                if (response.getBitmap() != null) {
                    view.setImageBitmap(response.getBitmap());
                } else if (defaultImageResId != 0) {
                    view.setImageResource(defaultImageResId);
                }
            }
        };
    }</pre>
2、加载精确指定大小的本地或者网络图片，返回一个包含bitmap的ImageContainer。
<pre>/**
     * Issues a bitmap request with the given URL if that image is not available
     * in the cache, and returns a bitmap container that contains all of the data
     * relating to the request (as well as the default image if the requested
     * image is not available).
     * @param requestUrl The url of the remote image
     * @param imageListener The listener to call when the remote image is loaded
     * @param maxWidth The maximum width of the returned image.
     * @param maxHeight The maximum height of the returned image.
     * @return A container object that contains all of the properties of the request, as well as
     *     the currently available image (default if remote is not loaded).
     */
    public ImageContainer get(String requestUrl, ImageListener imageListener,
            int maxWidth, int maxHeight) {
        // only fulfill requests that were initiated from the main thread.
        throwIfNotOnMainThread();

        final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight);

        // Try to look up the request in the cache of remote images.
        Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
        if (cachedBitmap != null) {
            // Return the cached bitmap.
            ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

        // The bitmap did not exist in the cache, fetch it!
        ImageContainer imageContainer =
                new ImageContainer(null, requestUrl, cacheKey, imageListener);

        // Update the caller to let them know that they should use the default bitmap.
        imageListener.onResponse(imageContainer, true);

        // Check to see if a request is already in-flight.
        BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null) {
            // If it is, add this request to the list of listeners.
            request.addContainer(imageContainer);
            return imageContainer;
        }

        // The request is not already in flight. Send the new request to the network and
        // track it.
        Request&lt;?&gt; newRequest =
            new ImageRequest(requestUrl, new Listener() {
                @Override
                public void onResponse(Bitmap response) {
                    onGetImageSuccess(cacheKey, response);
                }
            }, maxWidth, maxHeight,
            Config.RGB_565, new ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    onGetImageError(cacheKey, error);
                }
            });

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey,
                new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;
    }</pre>
详细过程：先尝试从cache中获取bitmap，如果缓存命中，则直接显示图片。否则，先构造一个ImageContainer，显示默认图；再check一下是否存在重复请求，有的话将ImageContainer加入到request中以便请求返回后能够通知此ImageContainer；否则生成一个新的ImageRequest，加入RequestQueue（网络请求队列）和InFlightRequests（正在请求队列）。
3、图片下载和解析类ImageRequest，关键代码：
<pre>    @Override
    protected Response parseNetworkResponse(NetworkResponse response) {
        // Serialize all decode on a global lock to reduce concurrent heap usage.
        synchronized (sDecodeLock) {
            try {
                return doParse(response);
            } catch (OutOfMemoryError e) {
                VolleyLog.e("Caught OOM for %d byte image, url=%s", response.data.length, getUrl());
                return Response.error(new ParseError(e));
            }
        }
    }</pre>
parseNetworkResponse方法给解析bitmap加了一个同步锁，这样保证同一时刻只有一个线程在解析bitmap（请求下载图片是多线程的），可以减少同一时刻堆内存的占用，减少OOM的发生；万一堆内存确实不够，发生了OOM，由于捕获了异常，也不会导致程序crash，不过此时程序收到这个错误以后应该及时的释放一些内存以便能够继续加载图片。
下面是doParse方法的源码：
<pre>   /**
     * The real guts of parseNetworkResponse. Broken out for readability.
     */
    private Response doParse(NetworkResponse response) {
        byte[] data = response.data;
        BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
        Bitmap bitmap = null;
        if (mMaxWidth == 0 &amp;&amp; mMaxHeight == 0) {
            decodeOptions.inPreferredConfig = mDecodeConfig;
            bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
        } else {
            // If we have to resize this image, first get the natural bounds.
            decodeOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
            int actualWidth = decodeOptions.outWidth;
            int actualHeight = decodeOptions.outHeight;

            // Then compute the dimensions we would ideally like to decode to.
            int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,
                    actualWidth, actualHeight);
            int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,
                    actualHeight, actualWidth);

            // Decode to the nearest power of two scaling factor.
            decodeOptions.inJustDecodeBounds = false;
            // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
            // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
            decodeOptions.inSampleSize =
                findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
            Bitmap tempBitmap =
                BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

            // If necessary, scale down to the maximal acceptable size.
            if (tempBitmap != null &amp;&amp; (tempBitmap.getWidth() &gt; desiredWidth ||
                    tempBitmap.getHeight() &gt; desiredHeight)) {
                bitmap = Bitmap.createScaledBitmap(tempBitmap,
                        desiredWidth, desiredHeight, true);
                tempBitmap.recycle();
            } else {
                bitmap = tempBitmap;
            }
        }

        if (bitmap == null) {
            return Response.error(new ParseError(response));
        } else {
            return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
        }
    }</pre>
首先，如果MaxWidth和MaxHeight都为0的话，就按照原始大小解析bitmap；否则先解析得到原始图片的大小actualWidth和actualHeight，然后加上最大宽高的限制以后得到一个desiredWidth和desiredHeight，再根据这两个与actualWidth和actualHeight成比例的值计算出inSampleSize（这个inSampleSize会保证压缩得到的图片宽高均大于或等于desiredWidth和desiredHeight），解析得到bitmap后，如果发现实际的bitmap的长和宽大于期望的长宽，则将其缩放至期望的长宽（由于期望和实际的长宽是等比例的，所有压缩是等比例的），最后返回bitmap或者错误信息。
4、取消下载图片
<pre>       /**
         * Releases interest in the in-flight request (and cancels it if no one else is listening).
         */
        public void cancelRequest() {
            if (mListener == null) {
                return;
            }

            BatchedImageRequest request = mInFlightRequests.get(mCacheKey);
            if (request != null) {
                boolean canceled = request.removeContainerAndCancelIfNecessary(this);
                if (canceled) {
                    mInFlightRequests.remove(mCacheKey);
                }
            } else {
                // check to see if it is already batched for delivery.
                request = mBatchedResponses.get(mCacheKey);
                if (request != null) {
                    request.removeContainerAndCancelIfNecessary(this);
                    if (request.mContainers.size() == 0) {
                        mBatchedResponses.remove(mCacheKey);
                    }
                }
            }
        }</pre>
检查当前图片网络请求集合或者图片结果分发集合，如果含有此ImageRequest的话就取消。取消的代码如下：
<pre>       /**
         * Detatches the bitmap container from the request and cancels the request if no one is
         * left listening.
         * @param container The container to remove from the list
         * @return True if the request was canceled, false otherwise.
         */
        public boolean removeContainerAndCancelIfNecessary(ImageContainer container) {
            mContainers.remove(container);
            if (mContainers.size() == 0) {
                mRequest.cancel();
                return true;
            }
            return false;
        }</pre>
5、图片结果分发
<pre>   /**
     * Starts the runnable for batched delivery of responses if it is not already started.
     * @param cacheKey The cacheKey of the response being delivered.
     * @param request The BatchedImageRequest to be delivered.
     * @param error The volley error associated with the request (if applicable).
     */
    private void batchResponse(String cacheKey, BatchedImageRequest request) {
        mBatchedResponses.put(cacheKey, request);
        // If we don't already have a batch delivery runnable in flight, make a new one.
        // Note that this will be used to deliver responses to all callers in mBatchedResponses.
        if (mRunnable == null) {
            mRunnable = new Runnable() {
                @Override
                public void run() {
                    for (BatchedImageRequest bir : mBatchedResponses.values()) {
                        for (ImageContainer container : bir.mContainers) {
                            // If one of the callers in the batched request canceled the request
                            // after the response was received but before it was delivered,
                            // skip them.
                            if (container.mListener == null) {
                                continue;
                            }
                            if (bir.getError() == null) {
                                container.mBitmap = bir.mResponseBitmap;
                                container.mListener.onResponse(container, false);
                            } else {
                                container.mListener.onErrorResponse(bir.getError());
                            }
                        }
                    }
                    mBatchedResponses.clear();
                    mRunnable = null;
                }

            };
            // Post the runnable.
            mHandler.postDelayed(mRunnable, mBatchResponseDelayMs);
        }
    }</pre>
在UI线程中顺序的遍历ImageContainer（含重复请求），调用相应的回调方法。
