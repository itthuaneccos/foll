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
