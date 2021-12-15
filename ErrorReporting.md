# 错误处理
GNU C库的一些函数会探测和上报一些错误情况，有时候你的程序需要检测这些错误情况，并做出对应的处理。

## 检测错误
大部分库函数会在他们处理失败的时候，返回一个特殊的值（-1、null指针或者类似于EOF的一个常量），不过这只能告诉你发生了错误，但是具体是哪种错误，就需要去检测定义在`errno.h`中的`errno`变量的值。

`errno`的值是一个错误码。你也可以自己更改`errno`的值。由于`errno`被生命为`volatile`，所以他是可以被信号处理器异步更改的。一个编写良好的信号处理器，都会正确的保存和恢复`errno`的值，所以一般情况下，你不需要担心这种情况，除非你是在写信号处理器。

程序启动的时候，`errno`被初始化为0。当库函数遇到错误后，会将errno设置成一个非0的值，来标识有错误发生。

所有的错误码都一个符号标识，它们都是定义在`errno.h`中的宏。所有的符号都是以E开头，加上一个大写单词或者数字。

所有的`errno`都是正整数且符号都不一样（例外是：`EWOULDBLOCK`和`EAGAIN`的值是一样的）。由于符号的值是互不相同的，所以可以在switch中把他们当做labels（但是前外别同时用EWOULDBLOCK和EAGAIN）。

`errno`没必要对应上那些符号值，因为有的库函数可能返回他们自己`errno`值（并不在符号列表里）。

## errno列表

参考[https://www.gnu.org/software/libc/manual/html_mono/libc.html#Error-Messages](https://www.gnu.org/software/libc/manual/html_mono/libc.html#Error-Messages)。这些符号都定义在errno.h中

## 错误信息

函数库设计了一些变量和函数，用来使你的程序更简单的用自定义的格式来报告有用的错误信息。

在`errlist.c`中，定义了`errno`和错误信息对应的数组。index是`errno`，value是错误信息。

```
#ifndef ERR_MAP
# define ERR_MAP(n) n
#endif

const char *const _sys_errlist_internal[] =
  {
#define _S(n, str)         [ERR_MAP(n)] = str,
#include <errlist.h>
#undef _S
  };
```

宏扩展后的结果是：
```
const char *const _sys_errlist_internal[] =
  {
    [0] = "Success",
    [1] = "Operation not permitted",
    ......
  };
```

这样 `_sys_errlist_internal` 就包含了errno和错误信息的对应关系了。

### Function: char * strerror (int errnum)

该函数是现在`strerror.c`中。
```
char *
strerror (int errnum)
{
  return __strerror_l (errnum, __libc_tsd_get (locale_t, LOCALE));
}
```

该函数只是`__strerror_l`的一个包装函数，具体实现在它里面。
```
/* Return a string describing the errno code in ERRNUM.  */
char *
__strerror_l (int errnum, locale_t loc)
{
  int saved_errno = errno;
  char *err = (char *) __get_errlist (errnum);
  if (__glibc_unlikely (err == NULL))
    {
      struct tls_internal_t *tls_internal = __glibc_tls_internal ();
      free (tls_internal->strerror_l_buf);
      if (__asprintf (&tls_internal->strerror_l_buf, "%s%d",
		      translate ("Unknown error ", loc), errnum) == -1)
	tls_internal->strerror_l_buf = NULL;

      err = tls_internal->strerror_l_buf;
    }
  else
    err = (char *) translate (err, loc);

  __set_errno (saved_errno);
  return err;
}
```

该函数首先保存了`errno`，防止自己修改原有的值，在结束的时候恢复该值。
```
  int saved_errno = errno;
......
  __set_errno (saved_errno);
```

然后通过errnum来获取到对应的错误信息
```
  char *err = (char *) __get_errlist (errnum);
```

对于拿到的错误信息，判断是否有预定义的信息。
* 如果有的话，就根据locale来转换成对应的国际化错误信息。
* 如果没有预定义的错误信息，就返回未知错误对应的国际化错误信息

上面是用到了`translate`函数来翻译错误信息
```
static const char *
translate (const char *str, locale_t loc)
{
  locale_t oldloc = __uselocale (loc);
  const char *res = _(str);
  __uselocale (oldloc);
  return res;
}
```

该函数先设置了locale，然后保存之前的locale的值。处理完后再恢复回去原先的值。

通过宏扩展来调用函数(dcgettext.c)
```
/* Look up MSGID in the DOMAINNAME message catalog for the current CATEGORY
   locale.  */
char *
DCGETTEXT (const char *domainname, const char *msgid, int category)
{
  return DCIGETTEXT (domainname, msgid, NULL, 0, 0, category);
}
```

其中category用的是5，定义来自于`locale.h`
```
#define __LC_CTYPE		 0
#define __LC_NUMERIC		 1
#define __LC_TIME		 2
#define __LC_COLLATE		 3
#define __LC_MONETARY		 4
#define __LC_MESSAGES		 5
#define __LC_ALL		 6
#define __LC_PAPER		 7
#define __LC_NAME		 8
#define __LC_ADDRESS		 9
#define __LC_TELEPHONE		10
#define __LC_MEASUREMENT	11
#define __LC_IDENTIFICATION	12
```

最后调用到(dcigettext.c)
```
/* Look up MSGID in the DOMAINNAME message catalog for the current
   CATEGORY locale and, if PLURAL is nonzero, search over string
   depending on the plural form determined by N.  */
#ifdef IN_LIBGLOCALE
char *
gl_dcigettext (const char *domainname,
	       const char *msgid1, const char *msgid2,
	       int plural, unsigned long int n,
	       int category,
	       const char *localename, const char *encoding)
#else
char *
DCIGETTEXT (const char *domainname, const char *msgid1, const char *msgid2,
	    int plural, unsigned long int n, int category)
#endif
{
#ifndef HAVE_ALLOCA
  struct block_list *block_list = NULL;
#endif
  struct loaded_l10nfile *domain;
  struct binding *binding;
  const char *categoryname;
  const char *categoryvalue;
  const char *dirname;
  char *xdirname = NULL;
  char *xdomainname;
  char *single_locale;
  char *retval;
  size_t retlen;
  int saved_errno;
  struct known_translation_t search;
  struct known_translation_t **foundp = NULL;
#if defined HAVE_PER_THREAD_LOCALE && !defined IN_LIBGLOCALE
  const char *localename;
#endif
  size_t domainname_len;

  /* If no real MSGID is given return NULL.  */
  if (msgid1 == NULL)
    return NULL;

#ifdef _LIBC
  if (category < 0 || category >= __LC_LAST || category == LC_ALL)
    /* Bogus.  */
    return (plural == 0
	    ? (char *) msgid1
	    /* Use the Germanic plural rule.  */
	    : n == 1 ? (char *) msgid1 : (char *) msgid2);
#endif

  /* Preserve the `errno' value.  */
  saved_errno = errno;

#ifdef _LIBC
  __libc_rwlock_define (extern, __libc_setlocale_lock attribute_hidden)
  __libc_rwlock_rdlock (__libc_setlocale_lock);
#endif

  gl_rwlock_rdlock (_nl_state_lock);

  /* If DOMAINNAME is NULL, we are interested in the default domain.  If
     CATEGORY is not LC_MESSAGES this might not make much sense but the
     definition left this undefined.  */
  if (domainname == NULL)
    domainname = _nl_current_default_domain;

  /* OS/2 specific: backward compatibility with older libintl versions  */
#ifdef LC_MESSAGES_COMPAT
  if (category == LC_MESSAGES_COMPAT)
    category = LC_MESSAGES;
#endif

  /* Try to find the translation among those which we found at
     some time.  */
  search.domain = NULL;
  search.msgid.ptr = msgid1;
  search.domainname = domainname;
  search.category = category;
#ifdef HAVE_PER_THREAD_LOCALE
# ifndef IN_LIBGLOCALE
#  ifdef _LIBC
  localename = __current_locale_name (category);
#  else
  categoryname = category_to_name (category);
#   define CATEGORYNAME_INITIALIZED
  localename = _nl_locale_name_thread_unsafe (category, categoryname);
  if (localename == NULL)
    localename = "";
#  endif
# endif
  search.localename = localename;
# ifdef IN_LIBGLOCALE
  search.encoding = encoding;
# endif

  /* Since tfind/tsearch manage a balanced tree, concurrent tfind and
     tsearch calls can be fatal.  */
  gl_rwlock_rdlock (tree_lock);

  foundp = (struct known_translation_t **) tfind (&search, &root, transcmp);

  gl_rwlock_unlock (tree_lock);

  if (foundp != NULL && (*foundp)->counter == _nl_msg_cat_cntr)
    {
      /* Now deal with plural.  */
      if (plural)
	retval = plural_lookup ((*foundp)->domain, n, (*foundp)->translation,
				(*foundp)->translation_length);
      else
	retval = (char *) (*foundp)->translation;

      gl_rwlock_unlock (_nl_state_lock);
# ifdef _LIBC
      __libc_rwlock_unlock (__libc_setlocale_lock);
# endif
      __set_errno (saved_errno);
      return retval;
    }
#endif

  /* See whether this is a SUID binary or not.  */
  DETERMINE_SECURE;

  /* First find matching binding.  */
#ifdef IN_LIBGLOCALE
  /* We can use a trivial binding, since _nl_find_msg will ignore it anyway,
     and _nl_load_domain and _nl_find_domain just pass it through.  */
  binding = NULL;
  dirname = bindtextdomain (domainname, NULL);
#else
  for (binding = _nl_domain_bindings; binding != NULL; binding = binding->next)
    {
      int compare = strcmp (domainname, binding->domainname);
      if (compare == 0)
	/* We found it!  */
	break;
      if (compare < 0)
	{
	  /* It is not in the list.  */
	  binding = NULL;
	  break;
	}
    }

  if (binding == NULL)
    dirname = _nl_default_dirname;
  else
    {
      dirname = binding->dirname;
#endif
      if (!IS_ABSOLUTE_PATH (dirname))
	{
	  /* We have a relative path.  Make it absolute now.  */
	  char *cwd = getcwd (NULL, 0);
	  if (cwd == NULL)
	    /* We cannot get the current working directory.  Don't
	       signal an error but simply return the default
	       string.  */
	    goto return_untranslated;
	  int ret = __asprintf (&xdirname, "%s/%s", cwd, dirname);
	  free (cwd);
	  if (ret < 0)
	    goto return_untranslated;
	  dirname = xdirname;
	}
#ifndef IN_LIBGLOCALE
    }
#endif

  /* Now determine the symbolic name of CATEGORY and its value.  */
#ifndef CATEGORYNAME_INITIALIZED
  categoryname = category_to_name (category);
#endif
#ifdef IN_LIBGLOCALE
  categoryvalue = guess_category_value (category, categoryname, localename);
#else
  categoryvalue = guess_category_value (category, categoryname);
#endif

  domainname_len = strlen (domainname);
  xdomainname = (char *) alloca (strlen (categoryname)
				 + domainname_len + 5);
  ADD_BLOCK (block_list, xdomainname);

  stpcpy ((char *) mempcpy (stpcpy (stpcpy (xdomainname, categoryname), "/"),
			    domainname, domainname_len),
	  ".mo");

  /* Creating working area.  */
  single_locale = (char *) alloca (strlen (categoryvalue) + 1);
  ADD_BLOCK (block_list, single_locale);


  /* Search for the given string.  This is a loop because we perhaps
     got an ordered list of languages to consider for the translation.  */
  while (1)
    {
      /* Make CATEGORYVALUE point to the next element of the list.  */
      while (categoryvalue[0] != '\0' && categoryvalue[0] == ':')
	++categoryvalue;
      if (categoryvalue[0] == '\0')
	{
	  /* The whole contents of CATEGORYVALUE has been searched but
	     no valid entry has been found.  We solve this situation
	     by implicitly appending a "C" entry, i.e. no translation
	     will take place.  */
	  single_locale[0] = 'C';
	  single_locale[1] = '\0';
	}
      else
	{
	  char *cp = single_locale;
	  while (categoryvalue[0] != '\0' && categoryvalue[0] != ':')
	    *cp++ = *categoryvalue++;
	  *cp = '\0';

	  /* When this is a SUID binary we must not allow accessing files
	     outside the dedicated directories.  */
	  if (ENABLE_SECURE && IS_PATH_WITH_DIR (single_locale))
	    /* Ingore this entry.  */
	    continue;
	}

      /* If the current locale value is C (or POSIX) we don't load a
	 domain.  Return the MSGID.  */
      if (strcmp (single_locale, "C") == 0
	  || strcmp (single_locale, "POSIX") == 0)
	break;

      /* Find structure describing the message catalog matching the
	 DOMAINNAME and CATEGORY.  */
      domain = _nl_find_domain (dirname, single_locale, xdomainname, binding);

      if (domain != NULL)
	{
#if defined IN_LIBGLOCALE
	  retval = _nl_find_msg (domain, binding, encoding, msgid1, &retlen);
#else
	  retval = _nl_find_msg (domain, binding, msgid1, 1, &retlen);
#endif

	  if (retval == NULL)
	    {
	      int cnt;

	      for (cnt = 0; domain->successor[cnt] != NULL; ++cnt)
		{
#if defined IN_LIBGLOCALE
		  retval = _nl_find_msg (domain->successor[cnt], binding,
					 encoding, msgid1, &retlen);
#else
		  retval = _nl_find_msg (domain->successor[cnt], binding,
					 msgid1, 1, &retlen);
#endif

		  /* Resource problems are not fatal, instead we return no
		     translation.  */
		  if (__builtin_expect (retval == (char *) -1, 0))
		    goto return_untranslated;

		  if (retval != NULL)
		    {
		      domain = domain->successor[cnt];
		      break;
		    }
		}
	    }

	  /* Returning -1 means that some resource problem exists
	     (likely memory) and that the strings could not be
	     converted.  Return the original strings.  */
	  if (__builtin_expect (retval == (char *) -1, 0))
	    break;

	  if (retval != NULL)
	    {
	      /* Found the translation of MSGID1 in domain DOMAIN:
		 starting at RETVAL, RETLEN bytes.  */
	      free (xdirname);
	      FREE_BLOCKS (block_list);
	      if (foundp == NULL)
		{
		  /* Create a new entry and add it to the search tree.  */
		  size_t msgid_len;
		  size_t size;
		  struct known_translation_t *newp;

		  msgid_len = strlen (msgid1) + 1;
		  size = offsetof (struct known_translation_t, msgid)
			 + msgid_len + domainname_len + 1;
#ifdef HAVE_PER_THREAD_LOCALE
		  size += strlen (localename) + 1;
#endif
		  newp = (struct known_translation_t *) malloc (size);
		  if (newp != NULL)
		    {
		      char *new_domainname;
#ifdef HAVE_PER_THREAD_LOCALE
		      char *new_localename;
#endif

		      new_domainname =
			(char *) mempcpy (newp->msgid.appended, msgid1,
					  msgid_len);
		      memcpy (new_domainname, domainname, domainname_len + 1);
#ifdef HAVE_PER_THREAD_LOCALE
		      new_localename = new_domainname + domainname_len + 1;
		      strcpy (new_localename, localename);
#endif
		      newp->domainname = new_domainname;
		      newp->category = category;
#ifdef HAVE_PER_THREAD_LOCALE
		      newp->localename = new_localename;
#endif
#ifdef IN_LIBGLOCALE
		      newp->encoding = encoding;
#endif
		      newp->counter = _nl_msg_cat_cntr;
		      newp->domain = domain;
		      newp->translation = retval;
		      newp->translation_length = retlen;

		      gl_rwlock_wrlock (tree_lock);

		      /* Insert the entry in the search tree.  */
		      foundp = (struct known_translation_t **)
			tsearch (newp, &root, transcmp);

		      gl_rwlock_unlock (tree_lock);

		      if (foundp == NULL
			  || __builtin_expect (*foundp != newp, 0))
			/* The insert failed.  */
			free (newp);
		    }
		}
	      else
		{
		  /* We can update the existing entry.  */
		  (*foundp)->counter = _nl_msg_cat_cntr;
		  (*foundp)->domain = domain;
		  (*foundp)->translation = retval;
		  (*foundp)->translation_length = retlen;
		}

	      __set_errno (saved_errno);

	      /* Now deal with plural.  */
	      if (plural)
		retval = plural_lookup (domain, n, retval, retlen);

	      gl_rwlock_unlock (_nl_state_lock);
#ifdef _LIBC
	      __libc_rwlock_unlock (__libc_setlocale_lock);
#endif
	      return retval;
	    }
	}
    }

 return_untranslated:
  /* Return the untranslated MSGID.  */
  free (xdirname);
  FREE_BLOCKS (block_list);
  gl_rwlock_unlock (_nl_state_lock);
#ifdef _LIBC
  __libc_rwlock_unlock (__libc_setlocale_lock);
#endif
#ifndef _LIBC
  if (!ENABLE_SECURE)
    {
      extern void _nl_log_untranslated (const char *logfilename,
					const char *domainname,
					const char *msgid1, const char *msgid2,
					int plural);
      const char *logfilename = getenv ("GETTEXT_LOG_UNTRANSLATED");

      if (logfilename != NULL && logfilename[0] != '\0')
	_nl_log_untranslated (logfilename, domainname, msgid1, msgid2, plural);
    }
#endif
  __set_errno (saved_errno);
  return (plural == 0
	  ? (char *) msgid1
	  /* Use the Germanic plural rule.  */
	  : n == 1 ? (char *) msgid1 : (char *) msgid2);
}
```

最终回到`/usr/share/locale/zh_TW/LC_MESSAGES/libc.mo`目录下找到对应的文件来进行翻译（zh_TW是locale的值）

### Function: char * strerror_r (int errnum, char *buf, size_t n)

该函数实际上调用的是`xpg-strerror.c`中的：
```
/* Fill buf with a string describing the errno code in ERRNUM.  */
int
__xpg_strerror_r (int errnum, char *buf, size_t buflen)
{
  const char *estr = __strerror_r (errnum, buf, buflen);

  /* We know that __strerror_r returns buf (with a dynamically computed
     string) if errnum is invalid, otherwise it returns a string whose
     storage has indefinite extent.  */
  if (estr == buf)
    return EINVAL;
  else
    {
      size_t estrlen = strlen (estr);

      /* Terminate the string in any case.  */
      if (buflen > 0)
	*((char *) __mempcpy (buf, estr, MIN (buflen - 1, estrlen))) = '\0';

      return buflen <= estrlen ? ERANGE : 0;
    }
}
```

该函数又调用了
```
char *
__strerror_r (int errnum, char *buf, size_t buflen)
{
  char *err = (char *) __get_errlist (errnum);
  if (__glibc_unlikely (err == NULL))
    {
      __snprintf (buf, buflen, "%s%d", _("Unknown error "), errnum);
      return buf;
    }

  return _(err);
}
```

该函数与`__strerror_l`的区别就是没有使用国际化、以及不会修改`errno`，所以并没有保存和回复`errno`的操作。

### Function: void perror (const char *message)
```
static void
perror_internal (FILE *fp, const char *s, int errnum)
{
  char buf[1024];
  const char *colon;
  const char *errstring;

  if (s == NULL || *s == '\0')
    s = colon = "";
  else
    colon = ": ";

  errstring = __strerror_r (errnum, buf, sizeof buf);

  (void) __fxprintf (fp, "%s%s%s\n", s, colon, errstring);
}


/* Print a line on stderr consisting of the text in S, a colon, a space,
   a message describing the meaning of the contents of `errno' and a newline.
   If S is NULL or "", the colon and space are omitted.  */
void
perror (const char *s)
{
  int errnum = errno;
  FILE *fp;
  int fd = -1;


  /* The standard says that 'perror' must not change the orientation
     of the stream.  What is supposed to happen when the stream isn't
     oriented yet?  In this case we'll create a new stream which is
     using the same underlying file descriptor.  */
  if (__builtin_expect (_IO_fwide (stderr, 0) != 0, 1)
      || (fd = __fileno (stderr)) == -1
      || (fd = __dup (fd)) == -1
      || (fp = fdopen (fd, "w+")) == NULL)
    {
      if (__glibc_unlikely (fd != -1))
	__close (fd);

      /* Use standard error as is.  */
      perror_internal (stderr, s, errnum);
    }
  else
    {
      /* We don't have to do any special hacks regarding the file
	 position.  Since the stderr stream wasn't used so far we just
	 write to the descriptor.  */
      perror_internal (fp, s, errnum);

      if (_IO_ferror_unlocked (fp))
	stderr->_flags |= _IO_ERR_SEEN;

      /* Close the stream.  */
      fclose (fp);
    }
}
```
该函数复用了`__strerror_r`来获取错误信息，然后打印到标准错误输出。

### Function: const char * strerrorname_np (int errnum)
该函数位于`strerrorname_np.c`中
```
const char *
strerrorname_np (int errnum)
{
  return __get_errname (errnum);
}
```

该函数只是`__get_errname`的包装
```

#ifndef ERR_MAP
# define ERR_MAP(n) n
#endif

static const union sys_errname_t
{
  struct
  {
#define MSGSTRFIELD1(line) str##line
#define MSGSTRFIELD(line)  MSGSTRFIELD1(line)
#define _S(n, str)         char MSGSTRFIELD(__LINE__)[sizeof(#n)];
#include <errlist.h>
#undef _S
  };
  char str[0];
} _sys_errname = { {
#define _S(n, s) #n,
#include <errlist.h>
#undef _S
} };

static const unsigned short _sys_errnameidx[] =
{
#define _S(n, s) \
  [ERR_MAP(n)] = offsetof(union sys_errname_t, MSGSTRFIELD(__LINE__)),
#include <errlist.h>
#undef _S
};

const char *
__get_errname (int errnum)
{
  if (errnum < 0 || errnum >= array_length (_sys_errnameidx)
      || (errnum > 0 && _sys_errnameidx[errnum] == 0))
    return NULL;
  return _sys_errname.str + _sys_errnameidx[errnum];
}
```

宏扩展后变为
```
static const union sys_errname_t
{
  struct
  {
char str3[sizeof("0")];
char str4[sizeof("EPERM")];
......
  };
  char str[0];
} _sys_errname = { {

"0",
"EPERM",
...
} };

static const unsigned short _sys_errnameidx[] =
{
[0] = offsetof(union sys_errname_t, str3),
[1] = offsetof(union sys_errname_t, str4),
......
};
```

这里面涉及到一些gcc中的宏处理：
```
__LINE__是当前行号

#define MSGSTRFIELD1(line) str##line

如果line是1，宏处理后就是str1

#define _S(n, str)         char MSGSTRFIELD(__LINE__)[sizeof(#n)];
如果n是1，当前行数是5，则处理后为：char str5[sizeof("1")]
```

可以自己编写测试文件来测试宏的展开，编写文件

`test.c`
```
# define ERR_MAP(n) n

static const union sys_errname_t
{
  struct
  {
#define MSGSTRFIELD1(line) str##line
#define MSGSTRFIELD(line)  MSGSTRFIELD1(line)
#define S(n, str)         char MSGSTRFIELD(__LINE__)[sizeof(#n)];
#include "test.h"
#undef S
  };
  char str[0];
} sys_errname = { {
#define S(n, s) #n,
#include "test.h"
#undef S
} };

static const unsigned short sys_errnameidx[] =
{
#define S(n, s) \
  [ERR_MAP(n)] = offsetof(union sys_errname_t, MSGSTRFIELD(__LINE__)),
#include "test.h"
#undef S
};
```

`test.h`
```
# define N_(msgid)      msgid
# define LOR 1
S(0, N_("Success"))
S(LOR, N_("No such file or directory"))

```

执行命令来查看预处理的结果
```
# gcc  -E test.c -o test.i
# cat test.i
```

## 备注

* 使用GNU 扩展功能时，需要在源码中`# define _GNU_SOURCE`或者在gcc选项中`-D_GNU_SOURCE`

## 参考

* [https://www.gnu.org/software/libc/manual/html_mono/libc.html](https://www.gnu.org/software/libc/manual/html_mono/libc.html)
