# Xamarin-Android
This is written at the time that I encounter some problems during my implementation

## The Orientation of an Image showed different from the one it was captured.

There are some circumstances that after I took a picture (ex. in vertical way) but when you see it's original file it appears horizontally as thumbnail.  
The data record in the **EXIF** seems to adjusted by **Auto-Rotate**  

At this moment if you just decode file as following you will get the wrong h/w and orientation.  

```
public Size GetImageSizeService(string imagePath)
{
    Bitmap imageBmp = BitmapFactory.DecodeFile(imagePath);
		return new Size((double)imageBmp.Width, (double)imageBmp.Height);	
}
```

The [ExifInterface](https://developer.xamarin.com/api/member/Android.Media.ExifInterface.GetAttributeInt/p/System.String/System.Int32/) stored the image information details in **Android.Media** including **ExifInterface.TagOrientation**.
![Image1](https://i.imgur.com/QXDw9NA.jpg)

The following reveal the bitmap in its correct orientation and given width / height.

```
 private Bitmap CheckRotateAndResizeImage(string imagePath, float width, float height)
{
    Bitmap originalImage = BitmapFactory.DecodeFile(imagePath);
    Bitmap resizedImage;
    ExifInterface exif = new ExifInterface(imagePath);

    int degree = 0;

    if (exif != null)
    {
        // Get TagOrientation
        int ori = exif.GetAttributeInt(ExifInterface.TagOrientation, (int)Orientation.Undefined);

        // Get degree
        switch (ori)
        {
            case (int)Orientation.Rotate90:
                degree = 90;
                break;
            case (int)Orientation.Rotate180:
                degree = 180;
                break;
            case (int)Orientation.Rotate270:
                degree = 270;
                break;
            default:
                degree = 0;
                break;
        }
    }
    if (degree != 0)
    {
        // Rotate Bitmap
        Matrix m = new Matrix();
        m.PostRotate((float)degree);
        originalImage = Bitmap.CreateBitmap(originalImage, 0, 0, originalImage.Width, originalImage.Height, m, true);
    }

    if (degree == 90 || degree == 270)
    {
        resizedImage = Bitmap.CreateScaledBitmap(originalImage, (int)height, (int)width, false);
    }

    else
    {
        resizedImage = Bitmap.CreateScaledBitmap(originalImage, (int)width, (int)height, false);
    }

    // Recycle Bitmap to Reduce Memory Usage
    if (originalImage != null)
    {
        originalImage.Recycle();
    }

    if (exif != null)
    {
        exif.Dispose();
    }

    return resizedImage;
}
```
