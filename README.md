# foll
#!/bin/bash
#
# When updating a branch, ensure it has the latest changes
# from other branches, e.g. stable.
#
# While this forces merging sooner than devs may like, it
# assures deployment and qa staff that the latest revisions
# they are qa'ing will always have the last stable release
# in it.
#
# Config
# ------
# hooks.update-ensure-follows.branches
#  Space-separated list of branches that other branches must merge with
# hooks.update-ensure-follows.excused
#  Space-separated list of branches that are excused from following (e.g. gitconfig)
#

. $(dirname $0)/functions

# Command line
refname="$1"
oldrev="$2"
newrev="$3"

# Look up the config variable and exit if not set
follows=$(git config hooks.update-ensure-follows.branches)
if [[ $? -ne 0 ]] ; then
	exit 0
fi

# Branch deletions are okay
if expr "$newrev" : '0*$' >/dev/null ; then
	exit 0
fi

# We only care about branches moving--ignore tags.
case "$refname" in
	refs/heads/*)
		short_refname=${refname##refs/heads/}
		;;
	*)
		exit 0
		;;
esac

excused=" $(git config hooks.update-ensure-follows.excused) "
if [[ $excused =~ " $short_refname " ]] ; then
	exit 0
fi

follows=($follows)
count=${#follows[@]}
for ((i = 0 ; i < count ; i++)) do
	follow="${follows[$i]}"
	git rev-parse --verify --quiet "$follow"
	if [ $? -eq 0 ] ; then
		missing_commits=$(git log ^$newrev $follow --pretty=oneline | wc -l)
		if [ $missing_commits -ne 0 ] ; then
			# If for some reason people are stupid and push with a --force flag,
			# we should warn them to update first in case one of their teammates
			# already merged for them
			if [ "0000000000000000000000000000000000000000" != "$oldrev" -a "$(git merge-base $oldrev $newrev)" != "$oldrev" ] ; then
				display_error_message "You need to update your local branch $short_refname"
			else
				display_error_message "You need to merge $follow into $short_refname"
			fi
			exit 1
		fi
	fi
done

exit 0
