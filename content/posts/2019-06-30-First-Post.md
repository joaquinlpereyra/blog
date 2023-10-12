---
layout: post
title: First Post
date: 2019-06-30 14:53:25
---

This will be an interesting one. Kind of testing at the moment.

I'm creating this blog with Hugo. I really really dislike Go's (and Hugo's) date handling. Specially when creating a post. Apparently one needs to MANUALLY set the date on a post? Why are we even programming really. Anyway, I created a small bash script to help me with that.

```
post() {
    shortdate=$(date "+%Y-%m-%d")
    date=$(date "+%Y-%m-%d %H:%M:%S")
    title=$1
    title_mod=${1// /-}
    title_mod=${title_mod//,/}
    categories=${2//,/ }
    # one should modify this, clearly
    post_path="$HOME/blog/content/post/$shortdate-$title_mod.md"
    if [[ ! -f $post_path ]]; then
        string="---\ntitle: $title\ndate: $date\ncategories: $categories\n---"
        echo "$string" > "$post_path"
    fi

    $EDITOR $post_path

    printf "Upload post? [N/y]: "
    read ans
    if [[ "$ans" == "y" ]]; then
        git add $post_path
        git commit -m "created post $title"
        # this is also probably something to change
        git push origin master
    fi
}
```
